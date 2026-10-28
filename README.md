# Allwinner D1: HiFi4 DSP + hardware spinlock test

Reproducible images and raw console logs for exercising the **sun6i hardware
spinlock** block on Allwinner D1, with the C906 running Linux and the Cadence
Tensilica **HiFi4** running FreeRTOS as the second bus master.

## Result

| | with lock | without lock |
|---|---|---|
| counter | **1 440 321** (exact) | 1 109 635 of 1 206 849 |
| lost updates | 0 | 97 214 (**8.1 %**) |
| tripwire / overlap | 0 / 0 | 29 872 Linux, 120 033 DSP |
| Linux iteration | 2 197 ns | 668 ns |
| verdict | **PASS** | **PASS** (control failed as expected) |

Linux runs a fixed 1 000 000 iterations and the DSP runs until Linux signals
stop, so how many it completes depends on how quickly Linux gets there: 440 321
in the locked run, 206 849 in the control, where it does not have to wait for a
lock it never takes. The check is therefore `counter == linux_iters + dsp_iters`,
not a fixed total.

With the lock the sum is exact and the tripwire — set while inside the critical
section and read by the other side — never fires. Without it, updates are lost
and overlap is observed on both sides.

The loss rate **varies between runs** (5.3 %, 5.7 %, 8.1 %, 17.3 %, 17.8 % and
18.7 % have been observed). Treat the percentage as one sample; the qualitative
result is what reproduces.

These numbers are `logs/2026-10-29-lichee-rv-dock.log`. The 2026-10-24 images
ran cacheless and are kept in `logs/2026-10-24-lichee-rv-dock.log` for
comparison: same verdicts, but a locked iteration cost 14 455 ns there, 6.6× the
figure above. That factor is the C906 caches (see
[The C906 caches](#the-c906-caches-and-maee)).

The lock is not free, but most of what it costs is not the lock. Measured
separately on the C906, 1 M iterations each, with the DSP polling:

| Step | caches off | caches on | change |
|---|---|---|---|
| bare register access: `readl()` to take, `writel()` to release | 882 ns | **126 ns** | 7× |
| the same plus the two `mb()` the hwspinlock core issues | 910 ns | **131 ns** | 7× |
| through `hwspin_lock_timeout()` + `hwspin_unlock()` | 8 342 ns | **487 ns** | 17× |
| the same pair on the DSP, bare metal (wall clock) | 478 ns | **453 ns** | — |

The hardware path is 0.13 µs and the memory barriers are essentially free; the
remaining ~0.35 µs is the Linux side — the call into the hwspinlock core, the
timeout bookkeeping and the core's own spinlock.

The DSP figure is the interesting one: it does **not** improve. A cross-core
handshake is bound by the DSP's turnaround and the bus round-trip, not by how
fast the C906 can read its own memory.

### The C906 caches and MAEE

The earliest images left both C906 caches disabled — `MHCR.IE` and `MHCR.DE`
reset to 0 and nothing in the boot chain (BROM, SPL, OpenSBI) set them, so every
normal-memory access cost ~150 ns and function calls were disproportionately
expensive. The images here enable both caches in the SPL, once DRAM is up.

Caches alone were not enough. This core has no Svpbmt, so the PTE carries no
memory-type attribute of its own, and with `MXSTATUS.MAEE` clear the MMU ignores
PTE[63:59] and uses the M-mode sysmap instead — a map that is not
software-programmable. So `ioremap()` and `dma_alloc_coherent()` kept handing out
cacheable write-back memory even with the caches on, and two workarounds were
needed to live with it: the test module maintained the shared block by hand
through the DMA API, and `sunxi-mmc` needed a write-back of its IDMAC descriptor
chain, because the IDMAC read stale descriptors — it transferred nothing and
raised no interrupt, and the first data transfer of an SD probe waited forever.

The SPL now sets `MXSTATUS.MAEE`, so the MMU takes the T-Head memory type out of
PTE[63:59] and Linux's `ERRATA_THEAD_MAE` alternative puts the non-cacheable
encoding there. `ioremap()` is then genuinely uncached and `dma_alloc_coherent()`
genuinely coherent, and both workarounds are gone: this kernel carries no
`sunxi-mmc` patch and the test module does no cache maintenance at all, only a
barrier at each handoff point. Nothing measurable changed when they went; the
numbers below are the same either way.

Effect on the Lichee RV Dock, `bench=4`, 1 M iterations:

| Probe | caches off | caches on |
|---|---|---|
| load from a hot 256-byte buffer | 148 ns | **6 ns** |
| register loop with no memory access at all | 147 ns | **6 ns** |
| `spin_lock_irqsave`/`spin_unlock_irqrestore` pair | 2 706 ns | **164 ns** |
| boot to `Freeing unused kernel image` | 5.98 s | **1.9 s** |

The shared block is uncached on the C906 side only. The DSP's `CA_BYPASS`
request for DDR does not take effect, so the firmware still keeps the block
coherent with its own `xthal_dcache_region_invalidate()`/`writeback()` calls.

`modprobe d1_hwlock_test bench=4 trials=1000000` reproduces the table; `bench=0`
(the default) is the contention test.

## Versions

| Component | Version | Branch |
|---|---|---|
| Board | Sipeed Lichee RV Dock (Allwinner D1-H, 512 MiB) | — |
| Linux | 7.3.0, torvalds `21a7f44fd076` + the 3 hwspinlock patches + the test module | [`ikruusa/linux` — `v7-4-21a7f44-hwspinlock`](https://github.com/ikruusa/linux/tree/v7-4-21a7f44-hwspinlock) |
| U-Boot | 2027.01-rc1 + the sun20i series with caches and MAEE + HiFi4 remoteproc | [`ikruusa/u-boot` — `v2027.01-rc1-d1-v1-dsp`](https://github.com/ikruusa/u-boot/tree/v2027.01-rc1-d1-v1-dsp) |
| OpenSBI | v1.9 | [`ikruusa/opensbi` — `sun20i-changes-v5`](https://github.com/ikruusa/opensbi/tree/sun20i-changes-v5) |
| FreeRTOS | HiFi4, firmware rev 8 | [`ikruusa/FreeRTOS-HIFI4-DSP` — `d1-hwlock-test`](https://github.com/ikruusa/FreeRTOS-HIFI4-DSP/tree/d1-hwlock-test) |
| Buildroot | gcc 16.2.0, binutils 2.46.1 | [`ikruusa/buildroot` — `d1-hwlock-v2`](https://github.com/ikruusa/buildroot/tree/d1-hwlock-v2) |
| Image | 126 877 696 B, sha256 `b9a1167c…` | — |

The `Buildroot -g<sha>-dirty` in the log's `Linux version` line is the
**toolchain's** build id, not this image's commit: gcc and binutils were built
from buildroot at that commit, and the `-dirty` suffix means uncommitted
defconfigs in the tree at the time. The two images here were built in different
trees ruled by different gcc builds, so the string differs between them; the
image itself comes from the branches above either way.

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

## Building from source

Nothing local is needed — all sources are public branches.

```
git clone -b d1-hwlock-v2 https://github.com/ikruusa/buildroot.git
cd buildroot
make sipeed_lichee_rv_dock_dsp_defconfig   # Lichee RV Dock
# or
make nezha_dsp_defconfig                   # Sipeed Nezha, 1 GB
make
```

The result is `output/images/sdcard.img`. The first build downloads ~320 MB and
builds the toolchain from scratch: about 17 minutes on 28 cores, 1.5–2 hours on
eight.

The two defconfigs pin Linux, U-Boot and OpenSBI to the branches in
[Versions](#versions) through their GitHub archive URLs, so the build fetches
exactly those trees, not whatever a local checkout happens to contain.

Buildroot regenerates U-Boot's kconfig only when its stamp is missing, so
**switching between the two defconfigs in one tree silently reuses the previous
board's U-Boot** — use a separate checkout, or `make uboot-dirclean` between
them.

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
`ext4load` → `rproc load` → `rproc start` works on a fresh power cycle. If
`rproc load` fails, `rproc list` distinguishes "no core found" from a load that
was rejected; neither is needed for a working run.

### 2. Boot Linux

```
=> boot
```

The DSP keeps running because the image's kernel command line carries
`clk_ignore_unused` (see [Caveats](#caveats)).

The root filesystem is mounted by `root=PARTUUID=00c0ffee-01`, not by
`/dev/mmcblk0p1`: the two MMC hosts on this SoC probe asynchronously and carry
no `mmc` aliases, so the SD card and the SDIO host swap `mmc0`/`mmc1` between
boots, and with `rootwait` a wrong guess never times out. The image's MBR disk
signature is pinned to `0xC0FFEE` in the genimage layout to make that name
stable. The same applies to anything else that would name the disk — use
`/dev/disk/by-partuuid/` rather than `mmcblk0`.

### 3. Run both controls

Log in as `root` (no password), then:

```
# mount -t debugfs none /sys/kernel/debug
# ls /sys/bus/platform/drivers/sun6i_hwspinlock/
# cat /sys/kernel/debug/sun6i_hwspinlock/supported
# modprobe d1_hwlock_test trials=1000000 dsp_timeout_ms=10000
# rmmod d1_hwlock_test
# modprobe d1_hwlock_test use_lock=0 trials=1000000 dsp_timeout_ms=10000
# rmmod d1_hwlock_test
# modprobe d1_hwlock_test bench=4
# rmmod d1_hwlock_test
```

The driver directory lists `3005000.hwlock` when the device tree node was found
and the driver bound to it — the quickest way to tell a missing node from a
test that cannot pass. The `supported` file reports how many locks the driver
found — 32 on the D1.

The negative control is mandatory: without it, a pass could come from a test
that structurally cannot fail. `bench=4` is optional and produces the cost table
under [The C906 caches](#the-c906-caches-and-maee). Expected verdicts and timings
are in [Result](#result); the console transcript is
`logs/2026-10-29-lichee-rv-dock.log`.

## Expected noise

None of these indicate a problem:

| Line | Why |
|---|---|
| `Starting network: … No such device` / `FAIL` | the dock overlay declares `wlan0`; no WiFi driver is built |
| `EXT4-fs (mmcblk0p1): INFO: recovery required on readonly filesystem` | first boot after flashing; the shipped image's journal is replayed once |
| `riscv: ELF compat mode unsupported` | `CONFIG_COMPAT` is off; unrelated to the DSP, which U-Boot loads |

## Provenance

The commit messages carry the reasoning, and several of them record traps that
cost real debugging time.

### Linux — the series under test

The branch is vanilla v7.4 plus this series and nothing else; the HDMI work that
shared its tree lives on separate branches and touches no file listed here.

| Commit | Note |
|---|---|
| [`35a4ed0d3d78c`](https://github.com/ikruusa/linux/commit/35a4ed0d3d78c) | driver: get the clock and the reset without names |
| [`968f3a27fb33c`](https://github.com/ikruusa/linux/commit/968f3a27fb33c) | dt-bindings: compatible for Allwinner D1 |
| [`bdc56ee62bed8`](https://github.com/ikruusa/linux/commit/bdc56ee62bed8) | DT node in `sunxi-d1-t113.dtsi` |
| [`eec0c7d007ee2`](https://github.com/ikruusa/linux/commit/eec0c7d007ee2) | the contention test module, and the `bench=` mode that splits the lock cost into register, API and caller |

### U-Boot — the DSP loader

| Commit | Note |
|---|---|
| [`67ed62958a3`](https://github.com/ikruusa/u-boot/commit/67ed62958a3) | enable the C906 caches and MAEE in the SPL, once DRAM is up. MAEE is what makes the PTE memory type take effect, so `ioremap()` and `dma_alloc_coherent()` are after all these years genuinely uncached |
| [`65e05aa94bb`](https://github.com/ikruusa/u-boot/commit/65e05aa94bb) | RISC-V `memset_io()`, needed by the ELF loader |
| [`cf4c5606dd4`](https://github.com/ikruusa/u-boot/commit/cf4c5606dd4) | the HiFi4 remoteproc driver. It programs the boot vector **before** releasing reset: `DSP_CFG_RST` resets the CFG block too, so writes made earlier are silently lost |
| [`d33c72044d8`](https://github.com/ikruusa/u-boot/commit/d33c72044d8) | the firmware carveout on the dock — why `0x40900000` broke `booti` |
| [`9d89c916c2e`](https://github.com/ikruusa/u-boot/commit/9d89c916c2e) | the Nezha as a second DSP test board |
| [`e502c4696bb`](https://github.com/ikruusa/u-boot/commit/e502c4696bb) | the hwspinlock node (scaffolding) |

### OpenSBI

[`48ab44fe`](https://github.com/ikruusa/opensbi/commit/48ab44fe) D1s compatible ·
[`abe3628b`](https://github.com/ikruusa/opensbi/commit/abe3628b) reset via the reserved watchdog ·
[`8ad9f41e`](https://github.com/ikruusa/opensbi/commit/8ad9f41e) claim the whole page for the console UART

### FreeRTOS

[`13fb0ff`](https://github.com/ikruusa/FreeRTOS-HIFI4-DSP/commit/13fb0ff) — the DSP side of the contention test ·
[`209a52b`](https://github.com/ikruusa/FreeRTOS-HIFI4-DSP/commit/209a52b) firmware rev 6 — the DSP side of the benchmark, and no printing while it runs ·
[`bc9d87e`](https://github.com/ikruusa/FreeRTOS-HIFI4-DSP/commit/bc9d87e) firmware rev 8 — count executed iterations instead of the loop variable; under-reporting by one turned clean runs into FAIL

### Buildroot

[`4b188ac853`](https://github.com/ikruusa/buildroot/commit/4b188ac853) both DSP images build from the public branches instead of local tarballs ·
[`1746e551db`](https://github.com/ikruusa/buildroot/commit/1746e551db) firmware rev 8 in both overlays ·
[`babb594308`](https://github.com/ikruusa/buildroot/commit/babb594308) build from the cache-enabled branches ·
[`7ec69192c7`](https://github.com/ikruusa/buildroot/commit/7ec69192c7) image target ·
[`a5f207177b`](https://github.com/ikruusa/buildroot/commit/a5f207177b) branch-level pins ·
[`eba47cc7f4`](https://github.com/ikruusa/buildroot/commit/eba47cc7f4) Nezha image ·
[`3859f4b814`](https://github.com/ikruusa/buildroot/commit/3859f4b814) reuse the existing Nezha board files ·
[`c89db355d6`](https://github.com/ikruusa/buildroot/commit/c89db355d6) mount the rootfs by PARTUUID, first with per-target layouts ·
[`d06f0450bc`](https://github.com/ikruusa/buildroot/commit/d06f0450bc) pin the MBR disk signature in the shared layouts instead ·
[`2da929cf00`](https://github.com/ikruusa/buildroot/commit/2da929cf00) drop the per-target copies again ·
[`5523d3a00d`](https://github.com/ikruusa/buildroot/commit/5523d3a00d) firmware rev 6 and the `bench=` documentation ·
[`bc50c960a3`](https://github.com/ikruusa/buildroot/commit/bc50c960a3) repair the Nezha layout, whose `u-boot` partition the signature change had eaten

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
- **`0x1fe0c` is a hex size.** The size also feeds a sanity check against
  `e_shoff + sizeof(Elf32_Shdr)` (129 412 for this build), so a stale value is
  tolerated only until the firmware grows past it.

## License

The images are builds of GPL and LGPL components — Linux, U-Boot, OpenSBI,
BusyBox and Buildroot's userspace — from the public branches linked above. The
test module and DSP firmware live in those same branches and carry their trees'
licences.
