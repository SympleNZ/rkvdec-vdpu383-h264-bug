# rkvdec-vdpu383-h264 — RK3576 (VDPU383) H.264 deblocking corruption (fixed)

A mainline `rkvdec` H.264 decode bug on the Rockchip **RK3576** (VDPU383 IP): the decoder
intermittently corrupts the horizontal deblocking edges. **Root-caused and fixed** — the fix
is a small, self-contained, devicetree-free patch in [`fix/`](fix/).

> Reported to linux-media on 2026-05-15; root cause found and fixed 2026-06-06.
> Fix write-up: **[FIX.md](FIX.md)**. The investigation that led there is in
> [Investigation history](#investigation-history-superseded-by-the-fix) at the end.

## TL;DR

- **Symptom:** wrong pixels on luma rows `{4, 12, 13} mod 16` (the in-macroblock horizontal
  deblock edges), off by ~1, propagating through the following P-frames.
- **Scope:** RK3576 (VDPU383) only; **H.264 only** — HEVC and VP9 on the same IP are correct.
  **Non-deterministic**, with a board-dependent rate (from a few % up to ~40%).
- **Root cause:** the VDPU383 powers up with un-primed internal deblock-context state. The
  Rockchip BSP runs a one-shot priming ("warmup") decode at every power-up
  (`rk3576_workaround_run`) that the mainline V4L2 driver omits.
- **Fix:** run that priming decode at `pm_runtime_resume`. **64/64 bit-exact** vs `avdec_h264`,
  from a reproduced ~17–40% baseline race. Clean, devicetree-free patch:
  [`fix/0001-media-rkvdec-prime-VDPU383-deblock-warmup-rk3576.patch`](fix/0001-media-rkvdec-prime-VDPU383-deblock-warmup-rk3576.patch).
- **Not fixed by this:** throughput — mainline single-shot decode on RK3576 is slower than the
  BSP for *all* codecs (a separate limitation; see [FIX.md](FIX.md) § caveat).

## The fix

```sh
# in a mainline kernel tree (7.0 or 7.1 — the rkvdec driver is byte-identical in both):
git apply fix/0001-media-rkvdec-prime-VDPU383-deblock-warmup-rk3576.patch
# build + install (module or kernel), reboot, then confirm it is active:
dmesg | grep -i deblock-priming
#   expect: "RK3576 H.264 deblock-priming workaround enabled"
```

The patch adds one new file (`rkvdec-rk3576-workaround.c` — the priming buffer + the link-bank
kick sequence, ported from the BSP `rk3576_workaround_{init,run}`) and wires it from `rkvdec.c`
at probe (allocate once) + `pm_runtime_resume` (run on every power-up), scoped to the VDPU383
variant. **No devicetree change** — mainline already maps the `"link"` register bank the priming
uses. `git apply`-clean against v7.0 and v7.1-rc7.

If the `dmesg` line above is absent, the workaround did not build in or did not run — you are
testing the unpatched behaviour. (The fix is **not** self-contained in `rkvdec-vdpu383-h264.c`;
it needs this patch's probe + resume wiring, or the priming buffer is never allocated and the
warmup silently no-ops.)

Full root-cause analysis, the register kick sequence, and the validation matrix are in
**[FIX.md](FIX.md)**.

## Reproduce

```sh
# Software reference (decode once):
gst-launch-1.0 -q filesrc location=long.h264 \
  ! h264parse ! avdec_h264 ! videoconvert ! 'video/x-raw,format=NV12' \
  ! filesink location=sw.nv12

# Hardware decode — run SEVERAL times (the result is non-deterministic).
# Use a FULL decode (no num-buffers truncation); compare frame 0 each run:
for i in $(seq 1 12); do
  gst-launch-1.0 -q filesrc location=long.h264 \
    ! h264parse ! v4l2slh264dec ! 'video/x-raw,format=NV12' \
    ! filesink location=hw.nv12
  cmp -n 3110400 hw.nv12 sw.nv12 && echo "run $i: MATCH" || echo "run $i: differ"
done
```

On a broken (unpatched) stack, frame 0 **alternates** run-to-run between an exact match (0 bytes
differ) and a ~20% byte mismatch with fully-corrupted Y rows at `{4, 12, 13} mod 16` (pixels off
by ~1). Roughly ~80% of runs are correct and the rate varies, so a best-of-N or single short run
can look "clean" by luck — measure across many full decodes. On a broken run the IDR error
propagates and subsequent P-frames rise to ~25–27%.

## Sibling VDPU383 bugs

The same below-MMIO investigation toolkit developed here — register/GBL/CDF/stream byte-diffs vs
the vendor MPP stack, IOMMU access-trace fault probes, reserved-register-bit sweeps — has been
applied to the two sibling VDPU383 codec bugs. H.264 turned out to be the **fixable** one
(un-primed power-up state → warmup). The other two are deeper and remain open below the MMIO
interface:

- **VP9 compound (SELECT / alt-ref)** — [`SympleNZ/rkvdec-vdpu383-vp9`](https://github.com/SympleNZ/rkvdec-vdpu383-vp9):
  the complete entropy-input prob buffer is byte-identical to MPP (0 diffs), the candidate
  references are fetched, and no reserved register bit gates it — yet the HW decodes zero compound
  blocks (`comp_mode` never adapts). The divergence is the per-block compound/single **decision**,
  upstream of the (proven-sound) averaging unit.
- **AV1 partial decode** — [`SympleNZ/rkvdec-vdpu383-av1`](https://github.com/SympleNZ/rkvdec-vdpu383-av1):
  the HW writes the intra above-row context; the failure scales with **content** (0–65% of rows
  survive), not a fixed row count — a content-driven internal-state exhaustion.

## Hardware & files

- **SoC:** Rockchip RK3576 (VDPU383). Decoder ID `readl(0x27b00100) = 0x38321746` on every RK3576
  board tested (NanoPi R76S, ArmSoM Sige5, Radxa Rock 4D).

| File | Size | Description |
|------|------|-------------|
| `long.h264` | 21 MB | Test input: 600 frames, 1920×1080 H.264, `openh264enc` SMPTE bars. |
| `*_boot.bin` | 16 MiB ×3 | First 16 MiB of the boot devices (Armbian-broken vs FriendlyElec-BSP-clean). Captured while investigating a firmware/BL31 angle — see the note below. |

> **BL31 / firmware is *not* the cause — it was a confound.** The bug correlated with the boot
> stack (Armbian-broken vs BSP-clean), which initially implicated BL31/DDR firmware. The real
> cause is the missing power-up warmup; the BSP stack merely *pairs* that warmup with an older
> BL31, so the firmware-version correlation is coincidental. The bootloader dumps and the
> firmware inventory are retained in [Investigation history](#firmware-inventory-bl31-ruled-out)
> as the board record, not as a causal claim.

## Status

H.264 correctness: **fixed** by this repo's patch. The patch is a clean, devicetree-free,
mainline-applicable candidate; it (or an equivalent) is the natural fix for the in-tree driver.
Throughput on RK3576 (all codecs) is a separate, still-open mainline limitation.

---

# Investigation history (superseded by the fix)

The record below is how the bug was root-caused: the systematic elimination of candidate causes,
the controls, and the data that forced the "un-primed HW state" conclusion — which then led to the
BSP warmup and the fix in [FIX.md](FIX.md). Retained for the evidence trail; **superseded by the
fix above.**

> Headline that drove it: every *per-frame* input (registers, buffers, bitstream, submit model)
> was matched byte-for-byte to the working vendor MPP stack on the same silicon, yet it still
> raced. That pointed away from per-frame programming and toward a *device/session-level* setup
> MPP does once that mainline doesn't — which turned out to be the power-up priming decode.

## The bug is non-deterministic; isolated below the MMIO interface (2026-06-05)

Re-testing on current mainline (`rkvdec-vdpu383-h264.c`, kernel 7.0.1, NanoPi R76S) refined the
original report:

- **Non-deterministic, not a fixed ~20%.** Decoding the *same* input repeatedly, frame 0 flips
  run-to-run between bit-exact correct (0.00% diff) and ~20.57% deblocking corruption (wrong Y
  rows at `{4, 12, 13} mod 16`, off by ~1). Over 12 full decodes: typically ~10 correct, ~2
  broken; the rate itself varies between samples (17–33%). The original "deterministic ~20%" was
  sampling bias.
- **Measure with FULL decodes.** `num-buffers=1` / single-frame drain gives a different
  (degenerate) result; the real 0%/20.57% pattern needs ≥2 frames decoded. Pipeline depth
  modulates the probability slightly but never eliminates the race.

**Systematically eliminated as causes** (each A/B'd at N≈25–30 on hardware; none change the race):
RCB buffer *content* (0xAA/0x00 prefill — HW overwrites the filterd context and races on its own
data, not a stale read) · RCB sizes/geometry (MPP's exact captured values) · RCB column registers
· block auto-gating (reg10) · ctrl regs reg13/20/21 · AXI timing reg28/29 · decoder internal cache
clear (CACHE0 and CACHE0/1/2 BSP-style) · decoder clock rate (identical) · pipeline/buffer depth ·
pre-kick `wmb()` + `iommu_flush_iotlb_all()`.

**HEVC is the control.** HEVC shares the entire H.264 submit path (same `memcpy_toio` + single-shot
kick, same RCB allocator, same clocks) and is correct, while only H.264 races — so no shared-path
tweak can be the cause. The only H.264-specific variable left (after matching all register values
to MPP) is the HW deblock algorithm for H.264's **4-row** in-macroblock edges vs HEVC's 8-row.

**Submit model tested directly — also not the cause.** MPP issues the job via a link-table (CCU)
DMA-descriptor submit; mainline writes the register file via CPU MMIO and kicks single-shot. Wiring
the H.264 backend through a working link-table path and measuring head to head from a clean boot:
link-table 20/25 good vs single-shot 19/25 good — the same race at the same rate. The hazard is
independent of the submit model.

**Net:** a below-the-MMIO-interface HW behaviour in the H.264 deblock stage — not a missing/wrong
register, not the cache, not the submit model (all matched to MPP). Whatever the vendor stack does
to make the VDPU383 deblock H.264's 4-row edges deterministically is not visible at, or
controllable from, the mainline per-frame register/submit interface. → the lead became "what does
MPP set up *once* at device/session level, below per-frame programming?"

## RCB placement (SRAM vs DRAM) ruled out (2026-06-06)

An AV1-path finding showed the BSP rewrites the RCB base registers in-kernel at submit
(`mpp_set_rcbbuf`): SRAM only when the DT wires `rockchip,rcb-iova` and `frame_width >=
rcb_min_width`, else DRAM. The BSP board's DT has no `rcb-iova`, so MPP decodes with **DRAM** RCB
while mainline always uses **SRAM** — a blind spot in the earlier "registers match MPP" comparison
(the rewrite is after the HAL register dump). Forcing all-DRAM RCB in mainline and re-running:
clean 6/8 vs 5/8 broken (DRAM vs SRAM, N=8) — same race, same rate. RCB *placement* does not affect
the H.264 deblock race. (It *does* move AV1's separate intra-above-row bug, which is why it was
worth excluding.)

## RCB content-independence + warmup IRQ-reap probed (2026-06-08)

- **RCB data state is not the cause — shown directly.** Reading the filterd RCB (slot 6) after each
  decode and comparing across runs: its content — including the meaningful deblock-context head, not
  just an uninitialised tail — is non-deterministic run to run even across byte-identical *correct*
  decodes. GOOD and BAD runs both span all-unique CRCs with no shared "good state". So there is no
  stable working-vs-broken RCB state to compare; the buffer content does not determine correctness.
  This directly answers "dump the RCB before/after and compare against the working case" — the
  comparison is moot, the final content is downstream scratch. The power-up warmup primes HW
  *state*, not this buffer.
- **The standalone warmup raises no completion IRQ (so it can't be IRQ-reaped as-is).** Prototyping
  reaping the warmup by interrupt instead of the ~20 ms status poll: arming the link `INT_EN` before
  `CFG_DONE` and waiting on a completion — the warmup never raises an IRQ (tried bit 0, then all
  bits); it signals only via the `0x4c` status register, which is exactly why the BSP
  `rk3576_workaround_run` polls it. A genuinely IRQ-reaped warmup would have to ride a real task in
  the decode link table. The polled warmup is the shipping form.

## Firmware inventory (BL31 ruled out)

BL31/DDR firmware was investigated because the bug correlated with the boot stack; it is **not** the
cause (see the note in *Hardware & files* — the BSP pairs the warmup with an older BL31, so the
correlation is a confound). Retained as the board/firmware inventory:

| Stack | BL31 commit | fwver | DDR fw | Decode result |
|-------|-------------|-------|--------|---------------|
| Sige5 Armbian | v2.3-931-g4d7e811c6 | v1.20 | v1.08 | Broken — 20.57% Y bytes wrong on frame 0 |
| R76S Armbian | v2.3-931-g4d7e811c6 | v1.20 | v1.08 | Broken — 23.54% Y bytes wrong on frame 0 |
| R76S FriendlyElec BSP | (not surfaced) | v1.17 | v1.09 | Clean — 0 bytes differ on any of 10 frames |
| Reporter's Sige5 (Detlev Casanova) | v2.3-859-gc481e5368 | v1.14 | unknown | Clean — no reproduction |

The boot dumps in this repo (`sige5_armbian_emmc_boot.bin`, `r76s_sd_armbian_boot.bin`,
`r76s_emmc_bsp_boot.bin`) are the first 16 MiB of each board's boot device, retained as the record.
