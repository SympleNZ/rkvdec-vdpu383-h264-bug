# rkvdec-vdpu383-h264.c deblocking bug — reference data

Reference data accompanying the bug report:

> `[BUG] rkvdec-vdpu383-h264: wrong pixels at horizontal deblocking edges y=4 and y=12`
> linux-media@vger.kernel.org, 2026-05-15

This repo exists to give the kernel driver maintainers concrete artefacts to reproduce against — input file, bootloader binaries, nothing more.

## ✅ FIXED (correctness) 2026-06-06 — root cause found: missing RK3576 power-up warmup

The race is **not** a wrong register or a silicon lottery — it is **un-primed hardware
state after power-up**. The Rockchip BSP runs a one-shot priming decode
(`rk3576_workaround_run`, `hack/mpp_hack_rk3576.c`) at every decoder power-on
(probe + every `pm_runtime` resume) that mainline V4L2 omits. Replicating that warmup at
`pm_runtime_resume` **eliminates the deblock race**: 64/64 decodes bit-exact vs an
`avdec_h264` reference, vs a reproduced ~17–40% baseline race. **See [FIX.md](FIX.md)**
for the root cause, the register kick sequence, a reference port, and validation.

(Correctness only — mainline single-shot throughput is a separate limitation: real HDMI
playback still drops frames at 1080p30 / 4K is ~30 fps. See FIX.md §caveat.)

The original triage below is retained as the record that led here.

### Sibling VDPU383 bugs (2026-06-09 status)

The same below-MMIO investigation toolkit developed here — register/GBL/CDF/stream byte-diffs
vs MPP, IOMMU access-trace fault probes, and reserved-register-bit sweeps — has been applied to
the two sibling VDPU383 codec bugs. H.264 turned out to be the **fixable** one (un-primed
power-up state → warmup). The other two are deeper and remain open below the MMIO interface:

- **VP9 compound (SELECT/alt-ref)** — [`SympleNZ/rkvdec-vdpu383-vp9`](https://github.com/SympleNZ/rkvdec-vdpu383-vp9):
  the *complete* entropy-input prob buffer is byte-identical to MPP (0 diffs), the candidate
  references are fetched, and no reserved register bit gates it — yet the HW decodes zero
  compound blocks (`comp_mode` never adapts). The divergence is the per-block compound/single
  **decision**, upstream of the (proven-sound) averaging unit.
- **AV1 partial decode** — [`SympleNZ/rkvdec-vdpu383-av1`](https://github.com/SympleNZ/rkvdec-vdpu383-av1):
  the HW writes the intra above-row context; the failure scales with **content** (0–65% of rows
  survive), not a fixed row count — a content-driven internal-state exhaustion.

These confirm H.264's warmup fix was a *distinct, fixable* class; VP9/AV1 need the hardware
documentation.

## Update 2026-06-05 — the bug is NON-DETERMINISTIC; isolated to below-MMIO HW behaviour

Re-testing on current mainline (`rkvdec-vdpu383-h264.c`, kernel 7.0.1, NanoPi R76S)
substantially refines the original report:

- **It is non-deterministic, not a fixed ~20%.** From a freshly-booted board, decoding
  the *same* input repeatedly, frame 0 flips run-to-run between **bit-exact correct
  (0.00% diff)** and the **~20.57% deblocking corruption** (fully-wrong Y rows at
  `{4, 12, 13} mod 16`, pixels off by ~1). Over 12 full decodes: typically ~10 correct,
  ~2 broken (≈80% correct); the rate itself varies between samples (17–33% broken).
  Broken runs also differ slightly from each other. This independently corroborates the
  separately-reported run-to-run non-determinism (and the earlier "~10% of the time"
  observation). Our original "deterministic ~20%" was sampling bias.
- **Use FULL decodes to measure.** `num-buffers=1` / single-frame drain gives a
  *different* (degenerate) result; the real 0%/20.57% pattern needs ≥2 frames decoded.
  Pipeline depth modulates the probability slightly but never eliminates the race.

**Systematically eliminated as causes (each A/B'd at N≈25-30 on hardware; none change the
race):** RCB buffer *content* (0xAA/0x00 prefill — HW overwrites the filterd context and
races on its own data, not a stale read) · RCB sizes/geometry (set to MPP's exact
captured values reg152=38592/reg154=38592/reg156=49536/reg158=reg160=0) · RCB column
registers · block auto-gating (reg10) · ctrl regs reg13/20/21 · AXI timing reg28/29
(`addr_align`, `rd_latency`) · **decoder internal cache clear (CACHE0 and CACHE0/1/2
BSP-style, pre-kick)** · decoder clock rate (identical: aclk/core 594 MHz, hclk 198 MHz,
cabac 1 GHz) · pipeline/buffer depth · pre-kick `wmb()` + `iommu_flush_iotlb_all()`.

**HEVC is the control.** HEVC shares the *entire* H.264 submit path (same `memcpy_toio`
register write + `writel(DEC_ENABLE)` kick, same RCB allocator, no cache-clear/reset/
barrier, same clocks) and is **correct**, while only H.264 races. So no shared-path tweak
can be the cause — consistent with every negative above. The only H.264-specific variable
left (after matching all register values to MPP) is the HW deblock algorithm for H.264's
**4-row** in-macroblock edges vs HEVC's 8-row.

**Ground truth:** vendor MPP on the BSP stack (`mpi_dec_test`, same RK3576 silicon) is
bit-exact, and its per-decode register programming was captured and matched — i.e. the
**mainline V4L2 driver already programs MPP-equivalent registers**, yet still races.

**Submit model tested directly — also NOT the cause.** MPP/BSP issues the job via a
**link-table (CCU) DMA descriptor** submit; the mainline backend writes the register file
via CPU MMIO and kicks single-shot. We wired the H.264 backend into a working link-table
submit path (depth=1, descriptor-DMA, BSP-style completion) and measured the race head to
head from a clean boot: **link-table = 20/25 good (5 bad), single-shot = 19/25 good (6
bad)** — the *same* deblock race at the *same* rate. So the hazard is **independent of the
submit model**. (This matches separate VP9 work where the link path produced output
byte-identical to single-shot.)

**Net for maintainers:** this is a **below-the-MMIO-interface HW behaviour** in the H.264
deblock stage — not a missing/wrong register, not the cache, not the submit model (all
tested and matched to MPP). Whatever the vendor stack does to make the VDPU383 deblock
H.264's 4-row edges deterministically is not visible at, or controllable from, the
mainline V4L2 register/submit interface. Pointers to what MPP sets up *once* at the
device/session level (below per-frame programming) would be the most useful lead.

## Update 2026-06-06 — RCB placement (SRAM vs DRAM) also ruled out

A later finding on the AV1 path (same VDPU383 RCB allocator) showed the Rockchip BSP
kernel rewrites the RCB base registers *in the kernel at submit* (`mpp_set_rcbbuf`):
it uses SRAM only when the DT wires `rockchip,rcb-iova` and `frame_width >= rcb_min_width`,
otherwise leaving RCB in **DRAM**. The BSP board's DT has no `rcb-iova`, so the working
MPP stack decodes with **DRAM** RCB — while the mainline V4L2 driver always uses **SRAM**
RCB. That rewrite happens after the HAL register dump, so it was a blind spot in the
earlier "registers match MPP" comparison.

Forcing all-DRAM RCB in the mainline driver and re-running the deblock test changed
nothing: **clean 6/8 vs 5/8 broken (DRAM vs SRAM, N=8) — the same race at the same rate
and ~47% magnitude.** So RCB *placement*, like RCB content and geometry before it, does
not affect the H.264 deblock race. (The placement lever *does* move AV1's separate
intra-above-row bug — which is why it was worth excluding here; for H.264 it has no
effect.)

## Update 2026-06-08 — RCB content-independence shown directly; warmup IRQ-reap probed

Two follow-ups prompted by review feedback (is it an RCB *data state* problem; could the
warmup be reaped by IRQ instead of polled):

- **RCB data state is not the cause — shown directly.** Reading the filterd RCB (slot 6)
  *after* each decode and comparing across runs: its content — including the meaningful
  first-16 KB deblock-context head, not just an uninitialised tail — is **non-deterministic
  run to run even across byte-identical correct decodes**. GOOD and BAD runs both span
  all-unique buffer CRCs with no shared "good state" / "bad signature". So there is **no
  stable working-vs-broken RCB state to compare**, and the buffer content does not determine
  correctness. This is the direct answer to "dump the RCB before/after and compare against
  the working case": the comparison is moot because the final buffer content is downstream
  scratch, not the determinant. Confirms the content-independent read-before-write race
  (consistent with the 0xAA/0x00 prefill result above) that the power-up warmup — which
  primes HW *state*, not this buffer — is what fixes.

- **The standalone warmup raises no completion IRQ (so it can't be IRQ-reaped as-is).** We
  prototyped reaping the warmup by interrupt instead of the ~21 ms status poll: arm the link
  `INT_EN` before `CFG_DONE` and wait on a completion driven from the top-level IRQ handler.
  The warmup **never raises an IRQ** — every reap hit the timeout fallback (tried `INT_EN`
  bit 0, then all bits). The warmup descriptor is a CCU/link-init task, not a normal decode
  task; it signals only via the `0x4c` status register, which is exactly why the BSP
  `rk3576_workaround_run` polls it. Correctness was unaffected (the warmup still primes; the
  validation set stayed bit-exact). A genuinely IRQ-reaped warmup would have to be issued as
  a real task in the decode link table (so it rides the normal decode-completion IRQ) rather
  than as the standalone init descriptor. The polled warmup remains the shipping form.

## Hardware

- SoC: Rockchip RK3576 (VDPU383)
- Decoder ID `readl(0x27b00100)` = `0x38321746` across all RK3576 boards tested (NanoPi R76S, ArmSoM Sige5, Radxa Rock 4D)

## Files

| File | Size | Description |
|------|------|-------------|
| `long.h264` | 21 MB | Test input. 600 frames, 1920x1080 H.264, openh264enc SMPTE bars (`gst-launch-1.0 videotestsrc num-buffers=600 pattern=smpte ! 'video/x-raw,width=1920,height=1080,framerate=30/1' ! openh264enc ! h264parse ! filesink`). |
| `sige5_armbian_emmc_boot.bin` | 16 MiB | First 16 MiB of `/dev/mmcblk0` on ArmSoM Sige5. Armbian 26.2.0-trunk.896 trixie, kernel 7.0.6-edge-rockchip64. **BL31 v2.3-931-g4d7e811c6, fwver v1.20. Reproduces the bug.** |
| `r76s_sd_armbian_boot.bin` | 16 MiB | First 16 MiB of `/dev/mmcblk1` (SD) on NanoPi R76S. Armbian 26.2.0-trunk.896, kernel 7.0.1-edge-rockchip64. **BL31 v2.3-931-g4d7e811c6, fwver v1.20. Reproduces the bug.** |
| `r76s_emmc_bsp_boot.bin` | 16 MiB | First 16 MiB of `/dev/mmcblk0` (eMMC) on NanoPi R76S. FriendlyElec BSP, kernel 6.1.141. **BL31 fwver v1.17, DDR fw v1.09. Vendor MPP `mpi_dec_test` returns 0 byte diffs vs avdec_h264 SW reference on every frame.** |

## BL31 / firmware summary

| Stack | BL31 commit | fwver | DDR fw | Decode result |
|-------|-------------|-------|--------|---------------|
| Sige5 Armbian | v2.3-931-g4d7e811c6 | v1.20 | v1.08 (24/10/09, fcb0cfd52f) | Broken — 20.57% Y bytes wrong on frame 0 |
| R76S Armbian | v2.3-931-g4d7e811c6 | v1.20 | v1.08 (24/10/09, fcb0cfd52f) | Broken — 23.54% Y bytes wrong on frame 0 |
| R76S FriendlyElec BSP | (not surfaced in strings) | v1.17 | v1.09 (25/03/04, 2f85f4b2d4) | Clean — 0 bytes differ on any of 10 frames |
| Reporter's Sige5 (Detlev Casanova) | v2.3-859-gc481e5368 | v1.14 | unknown | Clean — no reproduction |

All Rockchip ATF builds carry the `derrick.huang` builder identity, indicating the same upstream Rockchip arm-trusted-firmware tree at different commits.

## Reproducer

The bug-report email and inline C reproducer live in the linux-media archive:

- Initial report: <https://lore.kernel.org/linux-media/> (search subject: `rkvdec-vdpu383-h264: wrong pixels at horizontal deblocking edges y=4 and y=12`)

To reproduce:

```sh
# Software reference (decode once)
gst-launch-1.0 -q filesrc location=long.h264 \
  ! h264parse ! avdec_h264 ! videoconvert ! 'video/x-raw,format=NV12' \
  ! filesink location=sw.nv12

# Hardware decode — run SEVERAL times (the result is non-deterministic).
# Use a FULL decode (no num-buffers truncation); compare frame 0 each run.
for i in $(seq 1 12); do
  gst-launch-1.0 -q filesrc location=long.h264 \
    ! h264parse ! v4l2slh264dec ! 'video/x-raw,format=NV12' \
    ! filesink location=hw.nv12
  cmp -n 3110400 hw.nv12 sw.nv12 && echo "run $i: MATCH" || echo "run $i: differ"
done
```

On a broken stack, expect frame 0 to **alternate** run-to-run between an exact match
(0 bytes differ) and a ~20.57% byte mismatch with fully-corrupted Y rows at row indices
`{4, 12, 13} mod 16` (pixels off by ~1). Roughly ~80% of runs are correct; the rate
varies. On a broken run, the IDR error propagates and subsequent P-frames rise to ~25-27%.
A best-of-N or single short run can therefore look "clean" by luck — measure across
many full decodes.
