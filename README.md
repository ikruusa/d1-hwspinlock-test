# Allwinner D1: HiFi4 DSP + hardware spinlock test

Reproducible images and raw console logs for exercising the **sun6i hardware
spinlock** block on Allwinner D1, with the C906 running Linux and the Cadence
Tensilica **HiFi4** running FreeRTOS as the second bus master.

Everything here is **test scaffolding**, not upstream material. The only piece
intended for submission is the driver series in the linux tree, linked under
[Provenance](#provenance).

## Result

| | with lock | without lock |
|---|---|---|
| counter | **2 000 000** (exact) | 1 885 761 of 2 000 000 |
| lost updates | 0 | 114 239 (**5.7 %**) |
| tripwire / overlap | 0 / 0 | 100 302 Linux, 674 915 DSP |
| Linux iteration | 14 692 ns | 3 278 ns |
| verdict | **PASS** | **PASS** (control failed as expected) |

Both masters incremented the same shared counter 1 000 000 times under lock 0.
With the lock the total is exact and the tripwire — set while inside the
critical section and read by the other side — never fires. Without it, updates
are lost and overlap is observed on both sides.

The loss rate **varies between runs** (5.3 %, 5.7 %, 17.3 % and 18.7 % have been
observed). Treat the percentage as one sample; the qualitative result is what
reproduces.

The lock is not free: about 11.4 µs per take/free on the C906, against ~3.3 µs
for the unprotected read-modify-write it protects.

## Versions

| Component | Version | Branch |
|---|---|---|
| Board | Sipeed Lichee RV Dock (Allwinner D1-H, 512 MiB) | — |
| Linux | 7.3.0-rc7 + 3 hwspinlock patches | `ikruusa/linux` — `d1-hwspinlock-test` |
| U-Boot | 2026.10-rc4 + DSP remoteproc | `ikruusa/u-boot` — `sun20i-dsp-v1` |
| OpenSBI | v1.9 | `ikruusa/opensbi` — `sun20i-changes-v5` |
| FreeRTOS | HiFi4, firmware rev 4 | `ikruusa/FreeRTOS-HIFI4-DSP` — `d1-hwlock-test` |
| Buildroot | gcc 16.2.0, binutils 2.46.1 | `ikruusa/buildroot` — `sun20i-dsp-v1` |
| Image | 126 877 696 B, sha256 `2a64be76…` | — |

The kernel identifies itself as `Buildroot -g3859f4b814`, the buildroot commit
above, visible in the log's `Linux version` line.

## Prebuilt images

Both images are committed bzip2-compressed (127 MB → 7.98 MB):

| File | Board | Image sha256 | Status |
|---|---|---|---|
| `images/sdcard-dock-dsp.img.bz2` | Lichee RV Dock | `2a64be76…` | **tested on hardware** |
| `images/sdcard-nezha-dsp.img.bz2` | Sipeed Nezha, 1 GB | `1b5001a7…` | built, **never booted** |

```
sha256sum -c SHA256SUMS                            # the archives
bunzip2 -k images/sdcard-dock-dsp.img.bz2          # -> 126877696 bytes
sha256sum -c --ignore-missing SHA256SUMS.images    # the extracted images
dd if=images/sdcard-dock-dsp.img of=/dev/sdX bs=1M conv=fsync
```

`--ignore-missing` lets you verify one board without extracting the other.

Write the **whole disk**, not a partition. There is no FAT partition: the SPL
lives at 8 KiB and the rootfs starts at 1 MiB.

## Building from source

Nothing local is needed — all sources are public branches.

```
git clone -b sun20i-dsp-v1 https://github.com/ikruusa/buildroot.git
cd buildroot
make sipeed_lichee_rv_dock_dsp_defconfig   # Lichee RV Dock
# or
make nezha_dsp_defconfig                   # Sipeed Nezha, 1 GB
make
```

The result is `output/images/sdcard.img`. The first build downloads ~320 MB and
builds the toolchain from scratch: about 17 minutes on 28 cores, 1.5–2 hours on
eight.

Buildroot regenerates U-Boot's kconfig only when its stamp is missing, so
**switching between the two defconfigs in one tree needs `make uboot-dirclean`**
— otherwise you silently get the previous board's U-Boot.

## Running the test

Console: UART0, 115200 8N1.

### 1. Load the firmware from U-Boot

Interrupt the 2-second countdown ("Hit any key to stop autoboot"), then:

```
=> ext4load mmc 0:1 0x42000000 /root/dsp/dsp.elf
130292 bytes read in 17 ms (7.3 MiB/s)
=> rproc init
=> rproc list
0 - Name:'dsp0' type:'internal memory mapped' supports: load start stop reset is_running
=> rproc load 0 0x42000000 0x1fcf4
Load Remote Processor 0 with data@addr=0x42000000 130292 bytes: Success!
=> rproc start 0
```

The DSP announces itself on the same UART:

```
 FreeRTOS for HIFI4 DSP v0.1.1, Build on xtensa-hifi4-elf-gcc
==========================================================================
vTaskStartScheduler
hwlock: fw rev 4, shared @ 0x483ff000, stack top 0x48015f60
```

`0x1fcf4` = 130292, the byte count `ext4load` prints. **`rproc load` parses its
size as hex**, so the decimal figure cannot be used directly.

`rproc init` is not strictly required — `rproc load` probes the device on
demand — but `rproc list` turns "load failed" into "no core found", which is the
more useful error to see first.

### 2. Boot Linux

```
=> boot
```

The DSP keeps running because the image's kernel command line carries
`clk_ignore_unused` (see [Caveats](#caveats)).

### 3. Run both controls

Log in as `root` (no password):

```
# ls /sys/bus/platform/drivers/sun6i_hwspinlock/
3005000.hwlock  module          unbind
bind            uevent

# mount -t debugfs none /sys/kernel/debug
# cat /sys/kernel/debug/sun6i_hwspinlock/supported
32

# modprobe d1_hwlock_test trials=1000000 dsp_timeout_ms=10000
```

Takes about 15 s:

```
counter=2000000 expected=2000000 -> EXACT
tripwire linux=0 dsp=0 -> no overlap
linux burst 14692768 us (14692 ns/iter)
VERDICT: PASS
```

Then the negative control:

```
# rmmod d1_hwlock_test
# modprobe d1_hwlock_test use_lock=0
```

Takes about 3 s:

```
counter=1885761 expected=2000000 -> LOST UPDATES
tripwire linux=100302 dsp=674915 -> OVERLAP DETECTED
VERDICT: PASS (negative control failed as expected)
```

The negative control is mandatory: without it, a pass could come from a test
that structurally cannot fail.

## Expected noise

None of these indicate a problem:

| Line | Why |
|---|---|
| `Starting network: … No such device` / `FAIL` | the dock overlay declares `wlan0`; no WiFi driver is built |
| `ERROR: reserving fdt memory region failed (addr=48000000 …): -17` | `EEXIST` — U-Boot already reserved the carveout and `booti` tries again |
| `riscv: ELF compat mode unsupported` | `CONFIG_COMPAT` is off; unrelated to the DSP, which U-Boot loads |

## Provenance

The commit messages carry the reasoning, and several of them record traps that
cost real debugging time.

### Linux — the series under test

| Commit | Note |
|---|---|
| [`8bf0796a5c521`](https://github.com/ikruusa/linux/commit/8bf0796a5c521) | driver: get the clock and reset without names |
| [`16c9b5f1f8c7f`](https://github.com/ikruusa/linux/commit/16c9b5f1f8c7f) | dt-bindings: compatible for Allwinner D1 |
| [`db9ddbe1dffa4`](https://github.com/ikruusa/linux/commit/db9ddbe1dffa4) | DT node in `sunxi-d1-t113.dtsi` |
| [`423559a420391`](https://github.com/ikruusa/linux/commit/423559a420391) | the contention test module (scaffolding) |

### U-Boot — the DSP loader

| Commit | Note |
|---|---|
| [`f7c08ccac2f`](https://github.com/ikruusa/u-boot/commit/f7c08ccac2f) | RISC-V `memset_io()`, needed by the ELF loader |
| [`3ea72b0c2c1`](https://github.com/ikruusa/u-boot/commit/3ea72b0c2c1) | the HiFi4 remoteproc driver |
| [`ed2f0deedc6`](https://github.com/ikruusa/u-boot/commit/ed2f0deedc6) | **program the boot vector before releasing reset** — `DSP_CFG_RST` resets the CFG block, so writes made earlier are silently lost |
| [`0685f23cb11`](https://github.com/ikruusa/u-boot/commit/0685f23cb11) | hwlock node + DSP carveout — why `0x40900000` broke `booti` |
| [`8a4e03fcdff`](https://github.com/ikruusa/u-boot/commit/8a4e03fcdff) | Nezha defconfig, restored after being dropped upstream |

### OpenSBI

[`48ab44fe`](https://github.com/ikruusa/opensbi/commit/48ab44fe) D1s compatible ·
[`abe3628b`](https://github.com/ikruusa/opensbi/commit/abe3628b) reset via the reserved watchdog ·
[`8ad9f41e`](https://github.com/ikruusa/opensbi/commit/8ad9f41e) claim the whole page for the console UART

### FreeRTOS

[`13fb0ff`](https://github.com/ikruusa/FreeRTOS-HIFI4-DSP/commit/13fb0ff) — the DSP side of the contention test.

### Buildroot

[`7ec69192c7`](https://github.com/ikruusa/buildroot/commit/7ec69192c7) image target ·
[`a5f207177b`](https://github.com/ikruusa/buildroot/commit/a5f207177b) branch-level pins ·
[`eba47cc7f4`](https://github.com/ikruusa/buildroot/commit/eba47cc7f4) Nezha image ·
[`3859f4b814`](https://github.com/ikruusa/buildroot/commit/3859f4b814) reuse the existing Nezha board files

## Caveats

- **The DSP is loaded by U-Boot, not Linux.** There is no mainline D1 DSP
  remoteproc driver, so Linux only ever sees an already-running DSP.
- **`clk_ignore_unused` is required.** `ccu-sun20i-d1` registers `CLK_DSP` and
  `CLK_BUS_DSP_CFG` but nothing claims them, so `clk_disable_unused()` gates the
  DSP a few seconds into boot. It stops silently, with no fault flag set. A
  driver that owns those clocks is the real fix.
- **The DSP's DDR must be reserved `no-map`.** FreeRTOS links into DDR inside
  Linux's System RAM, so without the carveout the kernel allocates over the
  firmware's heap. The carveout must also stay out of `kernel_addr_r`'s path:
  placing it at `0x40900000` made `booti` fail outright, because U-Boot adds
  every `/reserved-memory` child to the LMB.
- **Do not read `0x00400000` from Linux.** Once the DSP owns its local SRAM,
  that access hangs the bus with no oops. Shared memory has to live in DDR;
  these tests use `0x483ff000`.
- **`0x1fcf4` is a hex size.** The size also feeds a sanity check against
  `e_shoff + sizeof(Elf32_Shdr)` (129 132 for this build), so a stale value is
  tolerated only until the firmware grows past it.

## License

The images are builds of GPL and LGPL components — Linux, U-Boot, OpenSBI,
BusyBox and Buildroot's userspace — from the public branches linked above. The
test module and DSP firmware live in those same branches and carry their trees'
licences.
