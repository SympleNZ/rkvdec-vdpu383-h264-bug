# rkvdec-vdpu383-h264.c deblocking bug — reference data

Reference data accompanying the bug report:

> `[BUG] rkvdec-vdpu383-h264: wrong pixels at horizontal deblocking edges y=4 and y=12`
> linux-media@vger.kernel.org, 2026-05-15

This repo exists to give the kernel driver maintainers concrete artefacts to reproduce against — input file, bootloader binaries, nothing more.

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
# Hardware decode (broken on BL31 v1.20)
gst-launch-1.0 -q filesrc location=long.h264 num-buffers=10 \
  ! h264parse ! v4l2slh264dec ! 'video/x-raw,format=NV12' \
  ! filesink location=hw.nv12

# Software reference
gst-launch-1.0 -q filesrc location=long.h264 num-buffers=10 \
  ! h264parse ! avdec_h264 ! videoconvert ! 'video/x-raw,format=NV12' \
  ! filesink location=sw.nv12

# Compare frame 0 (3,110,400 bytes per NV12 1080p frame)
cmp -n 3110400 hw.nv12 sw.nv12
```

On a broken stack, expect ~20-25% byte mismatch on frame 0 with fully-corrupted Y rows at row indices 4 and 12 mod 16. Rises to 25-27% on subsequent P-frames as the IDR error propagates.
