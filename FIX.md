# FIX — the deblock race is the missing RK3576 power-up warmup

**Status (2026-06-06): root cause found, correctness fix validated.** The
non-deterministic H.264 deblock corruption on `rkvdec-vdpu383-h264.c` (RK3576 / VDPU383)
is **not** a wrong register or a silicon lottery — it is an **un-primed hardware state**
after power-up. The Rockchip BSP runs a one-shot priming decode at every decoder power-on
that the mainline V4L2 driver does not. Replicating that warmup **eliminates the race**.

## Root cause

The BSP kernel driver runs `rk3576_workaround_run()`
(`drivers/video/rockchip/mpp/hack/mpp_hack_rk3576.c`, Rockchip, 2024). At init it builds a
tiny self-contained H.264 task in a DMA buffer — RLC slice + SPS/PPS + a full register set
(incl. `reg8` dec_mode, `reg13` timeout, `reg16` error-proc, `reg20/21` cabac-error,
`reg148` intra_rcb, `reg216` colmv) + a link descriptor — and **runs it through the
link/CCU path on every decoder power-up**: at probe and on every
`pm_runtime` **resume** (`mpp_rkvdec2.c`). It is a hardware **state initialisation**: it
primes the VDPU383's internal decode state after each power-on.

Mainline V4L2 `rkvdec` has **no equivalent** — it powers the IP up (`pm_runtime_resume`)
and decodes cold. The first decode(s) after each power-up therefore run with indeterminate
internal deblock state, which is exactly the observed **non-deterministic** corruption at
the H.264 4-row deblock edges (`{4,12,13} mod 16`), propagating through references.

This also explains every earlier negative in this repo: matching all per-frame registers,
RCB content/size/**placement**, clocks, caches, barriers, and the submit model to MPP
never helped, because the difference is **not per-frame** — it is the once-per-power-up
priming the vendor stack does and mainline omits.

## The fix

Run the RK3576 warmup at **`pm_runtime_resume`** (and probe), as the BSP does. The minimal
sufficient trigger is **resume** — it catches every power-up; running it only at
`start_streaming` left a ~10% residual (some decodes resume without a fresh stream start).

A self-contained, mainline-applicable patch is in this repo:
[`fix/0001-media-rkvdec-prime-VDPU383-deblock-warmup-rk3576.patch`](fix/0001-media-rkvdec-prime-VDPU383-deblock-warmup-rk3576.patch).
It adds a new `rkvdec-rk3576-workaround.c` (the priming buffer + the link-bank kick sequence,
ported from the BSP `rk3576_workaround_{init,run}`) and wires it from `rkvdec.c` at probe
(allocate once) + `pm_runtime_resume` (run on every power-up). It is **pure code, no
devicetree change** — mainline already maps the `"link"` register bank the priming uses.
Verified `git apply`-clean against v7.0 and v7.1-rc7 (the rkvdec driver is byte-identical
between them). The core kick sequence:

```c
/* link_base = the VDPU383 link MMIO bank; buf_iova = priming DMA buffer.
 * Descriptor is at buf+0x1000; priming H.264 task data at buf+0. */
writel(0x8000,  link_base + 0x58);                 /* ip_en: BIT(15) only      */
writel(0x7ffff, link_base + 0x54);                 /* ip watchdog              */
writel(0x10001, link_base + 0x00);                 /* ccu/init                 */
writel(lower_32_bits(buf_iova) + 0x1000, link_base + 0x04); /* CFG_ADDR -> desc */
writel(0x1, link_base + 0x08);                     /* LINK_MODE = 1            */
writel(0x1, link_base + 0x18);                     /* LINK_EN   = 1            */
wmb();                                              /* commit cfg before kick   */
writel(0x1, link_base + 0x0c);                      /* CFG_DONE -> HW commits   */
/* poll link_base+0x4c for status (~20 ms budget), then clear + tear down:    */
writel(0xffff0000, link_base + 0x48);
writel(0xffff0000, link_base + 0x4c);
writel(0, link_base + 0x00); writel(0, link_base + 0x08);
writel(0, link_base + 0x18); wmb(); writel(0, link_base + 0x58);
```

The priming task's register/data layout (offsets, the 32-byte RLC + SPS/PPS, the
descriptor) is taken verbatim from `mpp_hack_rk3576.c::rk3576_workaround_init`.

## Apply & test

```sh
# from your kernel tree (mainline 7.0 or 7.1 — rkvdec is identical in both):
git apply fix/0001-media-rkvdec-prime-VDPU383-deblock-warmup-rk3576.patch
# build + install the rkvdec module (or the kernel) as usual, then reboot.
```

**Confirm it is actually active — check this first:**
```sh
dmesg | grep -i "deblock-priming"
#   expect: "RK3576 H.264 deblock-priming workaround enabled"
```
If that line is absent, the workaround isn't compiled in or didn't run, and you're testing
the *unpatched* behaviour. (The usual cause is porting only the decode code without this
patch's probe hook — then the priming buffer is never allocated and the warmup silently
no-ops. The fix is **not** self-contained in `rkvdec-vdpu383-h264.c`; it needs this patch's
probe + resume wiring.)

**What to expect:** the rows-4/12/13 corruption gone. On our R76S this was 64/64 bit-exact
vs `avdec_h264` (baseline race ~17–40%). The race rate is **board-dependent**, so
confirmation across more RK3576 boards is welcome.

**What you do *not* need:** a devicetree change (mainline already maps `"link"`), or the
sibling VP9/AV1 out-of-tree module. This patch alone is the whole H.264 correctness fix.

**What it does *not* fix:** throughput (see the caveat below) — H.264, HEVC and VP9 on
mainline RK3576 are single-shot and slower than the BSP regardless of this patch.

> Status: **apply-verified** against v7.0 / v7.1-rc7. The underlying fix is validated 64/64
> in our development tree; build + on-hardware validation of *this extracted/cleaned patch*
> is in progress (independent confirmation welcome).

## Validation

Byte-exact vs `avdec_h264` software reference, on a fresh boot (NanoPi R76S, RK3576,
Armbian 7.0.1):

| stream | config | result |
|---|---|---|
| `bf.h264` (720p) | warmup OFF (baseline) | race present, ~17–40% of runs corrupt |
| `long.h264` (1080p, this repo) | warmup OFF | race present, 2/12 corrupt |
| `bf.h264` | warmup on **resume** only | **20/20 clean** |
| `bf.h264` + `long.h264` | warmup (all triggers) | **44/44 clean** |
| `bf.h264` | warmup **on by default**, no params | **20/20 clean** |

**Aggregate: 64/64 warmup decodes bit-exact; baseline race reproduced on both streams.**
The warmup runs once per power-up, so it does **not** regress throughput.

### Re-validated 2026-06-11 — two boards, A/B on demand

Note: the warmup's priming **persists** across runtime-suspend *and* module reload, so a clean
A/B can't be done mid-session — a true cold IP must be forced. Method: gate the warmup with a
`warmup_off` module param and power-cycle the IP via driver **unbind/rebind** before each arm.
HW decode vs a software reference (openh264dec / known-correct), per clip:

| board | clip | WITHOUT warmup | WITH warmup |
|---|---|---|---|
| NanoPi R76S (7.0.1) | bbb1080 (real-world) | 11/60 (18%) corrupt | **0/40 clean** |
| NanoPi R76S | long.h264 (SMPTE) | erratic, 0–75% across cold cycles | 0/40 clean |
| ArmSoM Sige5 (7.0.6) | bbb1080 / long.h264 | **20/20 (100%) corrupt** | — (no build tree to load the warmup) |

The rate is **board-dependent** (R76S ~18%, Sige5 100%) — consistent with the original
non-determinism — and the warmup **eliminates it (0/80 across both clips on the R76S)**. The
before/after frames in the README are the Sige5 (100%) HW decode vs the correct output.

## Caveat — this fixes correctness, not throughput

Mainline single-shot decode on RK3576 is throughput-limited independently of this bug:
real HDMI playback drops frames even at 1080p30, and 4K is ~30 fps vs the BSP's ~112 fps.
That is the mainline per-frame submit overhead (no link-mode pipelining), a separate issue
from the deblock race. The warmup makes H.264 **correct**; smooth real-time playback still
needs the link-mode/throughput work.

## TL;DR for the maintainer

The VDPU383 needs the BSP's `rk3576_workaround_run` priming decode at `pm_runtime_resume`;
mainline omits it; adding it eliminates the deblock race (64/64 bit-exact). A clean,
DT-free, mainline-applicable patch is in this repo:
[`fix/0001-media-rkvdec-prime-VDPU383-deblock-warmup-rk3576.patch`](fix/0001-media-rkvdec-prime-VDPU383-deblock-warmup-rk3576.patch)
— a new `rkvdec-rk3576-workaround.c` + a `pm_runtime_resume` hook, `git apply`-clean on
v7.0 / v7.1-rc7.
