# rkvdec-vdpu383-h264.c deblocking bug — reference data

Reference data accompanying the bug report:

> `[BUG] rkvdec-vdpu383-h264: wrong pixels at horizontal deblocking edges y=4 and y=12`
> linux-media@vger.kernel.org, 2026-05-15

This repo exists to give the kernel driver maintainers concrete artefacts to reproduce against — input file, bootloader binaries, nothing more.

## Update 2026-06-05 — the bug is NON-DETERMINISTIC; root cause narrowed to submit path

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

**Systematically eliminated as causes (each A/B'd at N≈30 on hardware; none change the
race):** RCB buffer *content* (0xAA/0x00 prefill — HW overwrites the filterd context and
races on its own data, not a stale read) · RCB sizes/geometry (set to MPP's exact
captured values reg152=38592/reg154=38592/reg156=49536/reg158=reg160=0) · RCB column
registers · block auto-gating (reg10) · ctrl regs reg13/20/21 · AXI timing reg28/29
(`addr_align`, `rd_latency`) · decoder clock rate (identical: aclk/core 594 MHz, hclk
198 MHz, cabac 1 GHz) · pipeline/buffer depth · pre-kick `wmb()` + `iommu_flush_iotlb_all()`.

**Ground truth:** vendor MPP on the BSP stack (`mpi_dec_test`, same RK3576 silicon) is
bit-exact, and its per-decode register programming was captured and matched — i.e. the
**mainline V4L2 driver already programs MPP-equivalent registers**, yet still races.

**Remaining difference = the submit path.** MPP/BSP issues the job via a **link-table
(CCU) DMA descriptor** submit; the mainline V4L2 backend writes the register file via CPU
MMIO (`memcpy_toio`) and kicks directly (single-shot). In separate VP9 work on the same
driver, switching VP9 to the existing link-table path sent *byte-identical* registers yet
changed decoded output — confirming the submit architecture (not the register values) is
the operative difference; but our V4L2 link path is incomplete (needs the BSP per-task
queue/lifecycle) and regressed VP9, so it is not a drop-in test for H.264.

**Net for maintainers:** this looks like a submit-mode / HW-pipeline timing hazard in the
deblock stage, not a missing or wrong register. The register programming is already
MPP-equivalent.

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
