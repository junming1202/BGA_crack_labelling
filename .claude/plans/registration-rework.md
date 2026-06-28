# Plan: 3.2c Registration Rework

## Goal
Replace the untrustworthy centroid/Hough registration in `02_image_registration.ipynb`
with a robust method, and prove its accuracy with the V1–V5 gates (V1–V3 must PASS).

## What the investigation found (evidence driving this plan)
1. **Color separates pads cleanly.** A pink/copper mask (`R>130 & R>B+20 & R>G+20`
   + morphology open/close) produces a near-perfect grid — far cleaner than grayscale
   Hough. ~1015 components (area>1500) vs the noisy 700–940 Hough circles.
2. **Most pads are dog-bones.** Each dumbbell = **two ball lobes** joined by a thin
   escape trace. 1015 dog-bone components × ~2 + singles ≈ 2077 balls. Only **4** pads
   are perfectly circular, so per-ball circularity cannot supply control points — this
   is exactly what corrupted the old centroid refinement (it took blob centroids of
   dog-bones and dye stains).
3. **Erosion splits the lobes.** Eroding the mask with a ~19 px ellipse breaks the thin
   bridges, yielding ~1100–1270 clean, round, isolated lobe blobs per image.
4. **Pitch is stable.** Projection-profile autocorrelation gives ~157–164 px pitch
   consistently across all images; rotation <1° (from prior V5).
5. **Why the old fit failed:** it RANSAC-fit an affine to only 9–25 points, several
   spurious → overfit. The fix is to feed it **hundreds of clean, well-spread control
   points** so the same affine math becomes robust.

## Approach (chosen: rebuild detection, keep global-affine fit but make it robust)

### New/changed notebook cells in `02_image_registration.ipynb`
Replace the detection + refinement helpers; keep the validation suite (§7).

1. **`pink_mask(im)`** — BGR color threshold for copper pads + morphology open/close.
2. **`detect_lobes(mask, erode_k=19, min_area=300)`** — erode to split dog-bones,
   `connectedComponentsWithStats`, filter by area, return lobe centroids (full-res).
   These are the clean control points (~1100–1270 per image, spread over the whole field).
3. **`fit_lattice_affine(centroids, coord_bv)`** — ICP-style global fit:
   - initial scale from projection-profile pitch (≈ pitch_px / 800);
   - 2–3 rounds of nearest-neighbour match (cKDTree) + `estimateAffine2D` RANSAC,
     tightening the threshold each round.
   - Returns `M`, inlier mask, and matched pairs. Expect **hundreds** of inliers.
   - Drop the old `centroid_refine` dye-blob step entirely.
4. **`register_image(...)`** rewritten to call the above; returns the same result-dict
   keys the validation cells already consume (`M`, `circles`→`centroids`, `refined_src/dst`,
   `inlier_mask`, `rmse`, etc.) so §7 works unchanged.

### Trustworthy accuracy metric (replaces the misleading inlier-only RMSE)
- **Primary metric:** full-field residual of ALL clean lobe centroids to their nearest
  projected grid node (median / p90 / max in px). This is independent of the fitting
  subset and covers the whole array.
- **Gate:** median residual < 5 px AND V1–V3 all PASS.
- V2 re-run on clean centroids (not raw Hough) — expected to clear ≥95%.
- V3 held-out CV on the large centroid set — expected held-out RMSE ≈ in-sample.
- V4/V5 already pass and remain as orientation/consistency guards.

### Visual confirmation (V1)
Regenerate `output/overlays/*.jpg` and a couple of high-res zoom crops; confirm predicted
dots sit centred on ball lobes across centre, edges, and corners.

## Acceptance criteria
- [ ] All 5 images: full-field median residual < 5 px (report p90/max too).
- [ ] V1 overlay: dots centred on lobes (centre + edges + corners), human-verified.
- [ ] V2 ≥ 95% of clean centroids within 0.15×pitch of a grid node.
- [ ] V3 held-out RMSE < 5 px and within ~2 px of in-sample.
- [ ] V4 + V5 still PASS.
- [ ] Notebook runs end-to-end clean.

## Then update docs
- AGENTS.md: flip 3.2 / 3.2c to done with the real metrics; update 4.2/4.2b gate results.
- Commit.

## Notes / risks
- Erosion kernel size (≈19 px) may need per-image robustness; if lobe counts vary, derive
  kernel from measured pitch rather than a constant.
- If a residual trend vs. position appears (lens distortion), consider a homography or
  per-region affine — but prior test showed homography ≈ affine, so affine is expected to suffice.
- Centroids only need to be many + clean + well-spread; we do NOT need to know each lobe's
  ball ID — the global affine handles assignment.
