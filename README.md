# Allwinner D1: HiFi4 DSP + hardware spinlock test

Reproducible images and raw console logs for exercising the **sun6i hardware
spinlock** block on Allwinner D1, with the C906 running Linux and the Cadence
Tensilica **HiFi4** running FreeRTOS as the second bus master. Both take the same
lock and hammer a shared counter; the questions are whether the count comes out
exact, and whether the two are ever inside the critical section at once.

## How the test works

The Linux side is `drivers/misc/d1-hwlock-test.c`, a module. The DSP side is a
FreeRTOS task in the firmware. Both take hardware lock 0 of the 32 the D1
exposes, and while holding it they increment a counter in a 56-byte structure at
`0x483ff000` — DDR inside the carveout that holds the firmware, so both sides are
looking at the same page and nothing else can be.

The tripwire is what turns a silent failure into a visible one. Each side writes
its name into a shared word on entering the critical section and clears it on
leaving, and reads the other's on the way in. A run where the counter is exact
and the tripwire never fires cannot have interleaved. A run with the lock
disabled should instead lose updates and show overlap on both sides — that is the
control, and it is what shows the test can fail at all.

Linux runs a fixed 1 000 000 iterations and the DSP runs until Linux signals
stop, so how far the DSP gets depends on contention rather than on a target. The
arithmetic that matters is `counter == linux_iters + dsp_iters`.

`bench=` measures the lock itself instead, with no contention, splitting the cost
into register access, the core's barriers, and the full API call.

## Result

| | with lock | without lock |
|---|---|---|
| counter | **1 440 321** (exact) | 1 109 635 of 1 206 849 |
| lost updates | 0 | 97 214 (**8.1 %**) |
| tripwire / overlap | 0 / 0 | 29 872 Linux, 120 033 DSP |
| Linux iteration | 2 197 ns | 668 ns |
| verdict | **PASS** | **PASS** (control failed as expected) |

The DSP completed 440 321 iterations in the locked run and 206 849 in the
control, where it does not have to wait for a lock it never takes.

With the lock the sum is exact and the tripwire never fires. Without it, updates
are lost and overlap is observed on both sides.

The loss rate **varies between runs** (5.3 %, 5.7 %, 8.1 %, 17.3 %, 17.8 % and
18.7 % have been observed). Treat the percentage as one sample; the qualitative
result is what reproduces.

These numbers are `logs/2026-10-29-lichee-rv-dock.log`.

## What the lock costs

`bench=4`, 1 M iterations:

| Step | ns/iteration |
|---|---|
| bare register access: `readl()` to take, `writel()` to release | 126 |
| the same plus the two `mb()` the hwspinlock core issues | 131 |
| through `hwspin_lock_timeout()` + `hwspin_unlock()` | 487 |
| the same pair on the DSP, bare metal (wall clock) | 453 |

So the lock itself is 0.13 µs and the two barriers add 5 ns. The other ~0.36 µs
of the 487 is Linux — the call into the hwspinlock core, the timeout bookkeeping
and the core's own spinlock. The DSP figure is the interesting one: it is no
faster than the C906's, because a cross-core handshake is bound by the DSP's
turnaround and the bus round-trip, not by how quickly either side reads its own
memory.

`modprobe d1_hwlock_test bench=4 trials=1000000` reproduces the table; `bench=0`
(the default) is the contention test.

## Versions

| Component | Version | Branch |
|---|---|---|
| Board | Sipeed Lichee RV Dock (Allwinner D1-H, 512 MiB) | — |
| Linux | 7.3.0, torvalds `21a7f44fd076` + the 3 hwspinlock patches + the test module | [`v7-4-21a7f44-hwspinlock`](https://github.com/ikruusa/linux/tree/v7-4-21a7f44-hwspinlock) |
| U-Boot | 2027.01-rc1 + the sun20i board support + the HiFi4 remoteproc driver | [`v2027.01-rc1-d1-v1-dsp`](https://github.com/ikruusa/u-boot/tree/v2027.01-rc1-d1-v1-dsp) |
| OpenSBI | v1.9 | [`sun20i-changes-v5`](https://github.com/ikruusa/opensbi/tree/sun20i-changes-v5) |
| FreeRTOS | HiFi4, firmware rev 8 | [`d1-hwlock-test`](https://github.com/ikruusa/FreeRTOS-HIFI4-DSP/tree/d1-hwlock-test) |
| Buildroot | gcc 16.2.0, binutils 2.46.1 | [`d1-hwlock-v2`](https://github.com/ikruusa/buildroot/tree/d1-hwlock-v2) |
| Image | 126 877 696 B, sha256 `b9a1167c…` | — |

The kernel branch is vanilla v7.4 plus this test and nothing else, so the result
is attributable to one commit plus the three-patch hwspinlock series. The commit
messages carry the reasoning for each patch, and several record traps that cost
real debugging time.

The `Buildroot -g<sha>-dirty` in the log's `Linux version` line is the
**toolchain's** build id, not this image's commit — gcc and binutils were built
from buildroot at that commit, and the `-dirty` suffix means uncommitted
defconfigs in the tree at the time.

## Prebuilt images

Both images are committed bzip2-compressed (127 MB → ~8 MB):

| File | Board | Image sha256 | Status |
|---|---|---|---|
| `images/sdcard-dock-dsp.img.bz2` | Lichee RV Dock | `b9a1167c…` | **tested on hardware** |
| `images/sdcard-nezha-dsp.img.bz2` | Sipeed Nezha, 1 GB | `63699015…` | built, **never booted** |

```
sha256sum -c SHA256SUMS                            # the archives
bunzip2 -k images/sdcard-dock-dsp.img.bz2          # -> 126877696 bytes
sha256sum -c --ignore-missing SHA256SUMS.images    # the extracted images
dd if=images/sdcard-dock-dsp.img of=/dev/sdX bs=1M conv=fsync
```

`--ignore-missing` lets you verify one board without extracting the other.

Write the **whole disk**, not a partition. There is no FAT partition: the SPL
lives at 8 KiB and the rootfs starts at 1 MiB.

## Building the image

Nothing local is needed — all sources are public branches.

```
git clone -b d1-hwlock-v2 https://github.com/ikruusa/buildroot.git
cd buildroot
make sipeed_lichee_rv_dock_dsp_defconfig   # Lichee RV Dock
# or
make nezha_dsp_defconfig                   # Sipeed Nezha, 1 GB
make
```

The result is `output/images/sdcard.img`. The first build builds the toolchain
from scratch: about 17 minutes on 28 cores, 1.5–2 hours on eight.

The two defconfigs pin Linux, U-Boot and OpenSBI to the branches in
[Versions](#versions) through their GitHub archive URLs, so the build fetches
exactly those trees and not whatever a local checkout happens to contain.

Buildroot regenerates U-Boot's kconfig only when its stamp is missing, so
switching between the two defconfigs in one tree silently reuses the previous
board's U-Boot — use a separate checkout, or `make uboot-dirclean` between them.

## Running the test

Console: UART0, 115200 8N1.

### 1. Load the firmware from U-Boot

Interrupt the 2-second countdown ("Hit any key to stop autoboot"), then:

```
=> ext4load mmc 0:1 0x42000000 /root/dsp/dsp.elf
=> rproc load 0 0x42000000 0x1fe0c
=> rproc start 0
```

The DSP announces itself on the same UART; `fw rev 8` is the build this test
needs (rev 6 and 7 predate the iteration-accounting fix and report a spurious
FAIL):

```
hwlock: fw rev 8, shared @ 0x483ff000, stack top 0x48016030
```

`0x1fe0c` is 130 572, the firmware size, and **`rproc load` parses its size as
hex** — the decimal figure printed by `ext4load` cannot be used directly. The
size changes whenever the firmware grows.

**A size that is too small is accepted silently.** `rproc load` does not compare
the argument with the file; a run with `0x1fcec` (130 284, 288 bytes short)
answered `Success!` just the same. Check that the byte count in its reply matches
the one `ext4load` printed.

`rproc init` is not needed — the uclass binds the device on first use, so
`ext4load` → `rproc load` → `rproc start` works on a fresh power cycle.

### 2. Boot Linux

```
=> boot
```

The DSP keeps running: the image's kernel command line carries `clk_ignore_unused`
(see [Caveats](#caveats)). The root filesystem is mounted by
`root=PARTUUID=00c0ffee-01` rather than by `/dev/mmcblk0p1`, because the two MMC
hosts on this SoC probe asynchronously and swap `mmc0`/`mmc1` between boots; the
image's MBR disk signature is pinned to make that name stable.

### 3. Run both controls

Log in as `root` (no password), then:

```
# mount -t debugfs none /sys/kernel/debug
# cat /sys/kernel/debug/sun6i_hwspinlock/supported
# modprobe d1_hwlock_test trials=1000000 dsp_timeout_ms=10000
# rmmod d1_hwlock_test
# modprobe d1_hwlock_test use_lock=0 trials=1000000 dsp_timeout_ms=10000
# rmmod d1_hwlock_test
# modprobe d1_hwlock_test bench=4
# rmmod d1_hwlock_test
```

The `supported` file reports how many locks the driver found — 32 on the D1. If
it reports nothing, `ls /sys/bus/platform/drivers/sun6i_hwspinlock/` shows whether
the device tree node bound.

The negative control is mandatory: without it, a pass could come from a test that
structurally cannot fail. `bench=4` is optional and produces the table under
[What the lock costs](#what-the-lock-costs). Expected verdicts are in
[Result](#result); the console transcript is
`logs/2026-10-29-lichee-rv-dock.log`.

## Expected noise

None of these indicate a problem:

| Line | Why |
|---|---|
| `Starting network: … No such device` / `FAIL` | the dock overlay declares `wlan0`; no WiFi driver is built |
| `EXT4-fs (mmcblk0p1): INFO: recovery required on readonly filesystem` | first boot after flashing; the shipped image's journal is replayed once |
| `riscv: ELF compat mode unsupported` | `CONFIG_COMPAT` is off; unrelated to the DSP, which U-Boot loads |

## Caveats

- **The DSP is loaded by U-Boot, not Linux.** There is no mainline D1 DSP
  remoteproc driver, so Linux only ever sees an already-running DSP.
- **Do not read `0x00400000` from Linux.** Once the DSP owns its local SRAM, that
  access hangs the bus with no oops. Shared memory has to live in DDR; these
  tests use `0x483ff000`.
- **`clk_ignore_unused` is required.** `ccu-sun20i-d1` registers the DSP clocks
  but nothing claims them, so `clk_disable_unused()` gates the DSP a few seconds
  into boot and it stops silently. The images here carry the parameter.

## License

The images are builds of GPL and LGPL components — Linux, U-Boot, OpenSBI,
BusyBox and Buildroot's userspace — from the public branches linked above. The
test module and DSP firmware live in those same branches and carry their trees'
licences.
