# Plan: Update docs/AGENTS.md

## What I Found (Research Summary)

### Existing data
| Item | Detail |
|---|---|
| Coordinate file | `database/FP11_BLTC/coordinates/BGA x,y coordinates FP11.xlsx`, sheet `BGA`, 2077 BGA balls |
| Coordinate columns | `BGA Number` (e.g. A10), `X Coord` (int µm), `Y Coord` (int µm) |
| Coordinate range | X ≈ ±17 450 µm, Y ≈ ±21 200 µm |
| Row 2 | Blank – skip when reading |
| Images | 5 × RGB JPEG, ~7 900–8 600 × 9 300–9 800 px, 20× optical magnification |

### Coordinate transform insight
The images are **bottom-view** (package reference corner = top-right of image), while the coordinate file is **top-down view** (reference corner = top-left). The required transform is a horizontal mirror:

```
x_image_space = -X_coord   (negate X)
y_image_space =  Y_coord   (Y unchanged)
```

Then scale from µm → pixels using known physical extents of the imaged area.

### Confirmed approach (from user)
- **CV MVP first** – build a working computer-vision baseline before adding AI/LLM assistance
- **Call out open questions** – document assumptions and validation gaps explicitly in AGENTS.md

---

## What Will Be Added to docs/AGENTS.md

The file currently has empty Steps 1–4. I will fill them in and add a **Assumptions & Open Questions** subsection near the top of Steps, keeping every other section unchanged.

### Section additions (in order)

#### Assumptions & Open Questions (new subsection before Step 1)
- Coordinate units are micrometers (µm); no unit field exists in the file
- Row 2 of the coordinate sheet is always blank
- No ground-truth labels exist yet; accuracy cannot be numerically validated without them
- The physical field-of-view of the 20× images is unknown; alignment must be derived from detected ball positions
- Crack percentage boundary rule: 0% = category A; ≥1% & ≤25% = B; ≥26% & ≤50% = C; ≥51% & ≤75% = D; ≥76% = E

#### Step 1: Research (fill in)
1.1 **IPC-TM-650-2.4.53 review** – Read `docs/ipc-tm-650-2.4.53 Dye and Pry.pdf` to understand what constitutes a crack vs normal dye penetration, and what visual features distinguish each category A–E.

1.2 **Coordinate system verification** – Manually locate 2–3 corner balls in a sample image and verify the ±X mirror transform produces the expected pixel positions.

1.3 **Sample image characterisation** – Note lighting, focus quality, dye contrast, any fiducial marks, and typical solder ball size in pixels.

1.4 **Manual labelling of ~20 balls** – Pick balls spanning different crack severities to understand visual difficulty before designing the classifier.

1.5 **Literature review (done)** – U-Net outperforms YOLO/ResNet for pixel-level crack segmentation (dice score 0.66). OpenCV SIFT + RANSAC homography is standard for sub-pixel image registration. YOLO-APGC achieves 99.2% on BGA surface defect detection.

#### Step 2: Plan (fill in)
**2.1 Pipeline overview**
```
coordinates.xlsx
  └─ parse + transform → bottom-view coords (µm)
                           └─ image registration → pixel coords per ball
                                                      └─ ROI extraction per ball
                                                           └─ crack segmentation (U-Net)
                                                                └─ area fraction → category A–E
                                                                     └─ <package>_BGA_label_output.xlsx
```

**2.2 Coordinate transform**
- Negate X: `x_bv = -X_coord`, keep Y: `y_bv = Y_coord`
- Scale µm → pixels: derive pixel-per-µm ratio from image registration (see 2.3)

**2.3 Image registration strategy**
- Detect solder ball centre positions in the image (Hough circle or template matching)
- Match detected centres to transformed coordinate grid (nearest-neighbour with RANSAC)
- Compute affine / homography transform; validate RMSE < 5 pixels on hold-out balls
- Fallback: manual corner annotation if automatic detection fails

**2.4 Crack segmentation approach**
- Use a pre-trained U-Net (e.g. `segmentation-models-pytorch`) fine-tuned or zero-shot on crack-like textures
- Each ball ROI (~100–200 px diameter) is independently classified
- Morphological post-processing to remove noise
- Crack fraction = crack pixels / ball interior pixels → map to A–E thresholds

**2.5 LLM assistance (optional, later)**
- Use the AMD gateway (`Claude-Opus-4.8`) only for edge-case validation or to assist with labelling difficult examples, not as the primary classifier
- Pass cropped ball image as base64 with a prompt asking for crack severity category

**2.6 Technology stack**
```
python       ≥3.11
uv           environment management
opencv-python image processing, Hough circles, SIFT, RANSAC
numpy, pandas data handling
openpyxl     read/write xlsx
torch + segmentation-models-pytorch  U-Net segmentation
openai       AMD LLM gateway (optional)
python-dotenv .env loading
jupyterlab   prototyping notebooks
```

**2.7 Output spec**
File: `database/<package>/output/<package>_BGA_label_output.xlsx`
Columns: `BGA Number`, `X Coord`, `Y Coord`, then one column per image named `<image_filename>` containing the category letter (A–E) for each ball.

#### Step 3: Build (fill in)
**Module layout**
```
src/
  core/
    coordinates.py    parse xlsx, apply mirror transform, return DataFrame
    registration.py   image → pixel transform (homography), quality metrics
    extraction.py     crop ROI per ball given transform + coords
    classifier.py     crack fraction → category A–E
    output.py         write output xlsx
  models/
    segmentation.py   U-Net wrapper (load weights, preprocess, infer, postprocess)
  utils/
    visualise.py      overlay ball positions on image, colour-code by category
  notebooks/
    01_coordinate_exploration.ipynb
    02_registration_prototype.ipynb
    03_segmentation_test.ipynb
    04_end_to_end_demo.ipynb
  main.py             CLI entry point: python main.py --package FP11_BLTC
```

**Implementation order** (each module depends on the previous)
1. `coordinates.py` – parse + transform
2. `registration.py` – derive pixel transform from detected balls
3. `extraction.py` – crop ROIs
4. `segmentation.py` – U-Net inference on ROIs
5. `classifier.py` – fraction → category
6. `output.py` – write xlsx
7. `main.py` – wire everything together
8. Notebooks – one per module for interactive prototyping

**Environment setup**
```bash
uv init
uv add opencv-python numpy pandas openpyxl torch segmentation-models-pytorch \
        openai python-dotenv jupyterlab
```
Copy `.env.example` to `.env` and set `LLM_GATEWAY_KEY=<your key>`.

#### Step 4: Test (fill in)
**4.1 Unit tests**
- Coordinate transform: verify corner balls map to expected quadrant
- Classification thresholds: check boundary values (0%, 1%, 25%, 26%, 50%, 76%, 100%)
- Output format: column names, row count matches coordinate file

**4.2 Registration quality gate**
- RMSE of ball-centre predictions vs detected centres < 5 px (target < 2 px for production)
- If RMSE > 5 px, flag the image and require manual correction

**4.3 End-to-end smoke test**
- Run `python main.py --package FP11_BLTC` on the 5 existing images
- Verify: output xlsx created, 2077 rows, 5 classification columns, no `NaN` values
- Visually inspect overlays for 10 random balls

**4.4 Manual accuracy check (requires ground-truth labels)**
- Manually classify 50 randomly selected balls across all 5 images
- Compare to automated labels; target ≥70% exact-match for MVP
- Document per-category confusion matrix

**4.5 Open validation questions** (explicit acknowledgement)
- No ground-truth labels exist yet; numerical accuracy cannot be computed until labelling is done
- U-Net may need fine-tuning on actual dye-and-pry images if zero-shot performance is poor
- Registration robustness across different packages (other than FP11) is unvalidated

---

## Changes to Existing Sections

**None.** Every existing section (header, dye-and-pry reference, folder structure, Goal, Environment setup, LLM setup code block, open-source model note) is preserved verbatim. Only the empty `### Step 1` through `### Step 4` bodies and a new `## Assumptions & Open Questions` subsection are added.

---

## Files Touched

| File | Change |
|---|---|
| `docs/AGENTS.md` | Fill in Steps 1–4; add Assumptions & Open Questions |

No source code, notebooks, or any other file is created in this PR — that happens in Step 3 (Build).
