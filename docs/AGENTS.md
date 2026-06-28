# BGA crack labelling tools
You are an agent that helps to develop a software tool that is able to automate the labelling of each BGA (Ball-grid array) solder ball crack severity into either one of the 5 different categories (category A: 0% crack, category B: 1% - 25% crack, category C: 26% - 50% crack, category D: 51% - 75% crack, category E: 76% - 100% crack).

## More info about dye and pry
@docs/ipc-tm-650-2.4.53 Dye and Pry.pdf

## General guidelines for files and folder structures
All the markdown documents like AGENTS.md will be located in "docs" folder.

All the python scripts and jupyter notebook will be located in "src" folder. Notebooks specifically are placed in `src/notebooks/`.

The "database" folder will contain a list of subfolders with specific name that user defines. For example, "FP11_BLTC" means a package called "FP11" that has undergoes "BLTC" (board-level temperature-cycling test). Within each subfolder, there are at least 2 folders called "images" and "coordinates".
"images" folder contains one or more images of the bottom-view of BGA package dye and pry results. Each image shows the BGA array that should match the coordinates for the package's coordinates file within "coordinates" folder. Note that in the dye and pry image within "images" folder, it shows the bottom view of a package so the reference point of the package is located at top right corner in the image while in the coordinates file, it shows the BGA coordinates of the package with top-down view so the reference point of the package is at top-left corner for coordinates file. Take note of this and apply any coordinates transformation as necessary later when processing the results.

## Goal
At the end of automation, an "output" folder will be created at the same directory as "coordinates" folder or "images" folder. In the "output" folder, an output file like "_BGA_label_output.xlsx" will be created with the name of parent directory of "coordinates" folder or "images" folder as prefix. The output file is a spreadsheet with copy info of BGA coordinates from "coordinates" folder and additional columns with each column contains the classification result (category A, B, C, D, E) of each BGA for each dye and pry image in "images" folder.      

## Environment setup
Please use uv for environment setup.

## Available resources 
### LLM Setup
Please refer to the python code below as an example to setup any required llm functionality. Strictly follow the settings of base_url = "https://llm-api.amd.com/Unified/v1", api_key = os.environ["LLM_GATEWAY_KEY"], "Ocp-Apim-Subscription-Key" = os.environ["LLM_GATEWAY_KEY"], model = "Claude-Opus-4.8".
```python
import os
from openai import OpenAI
from dotenv import load_dotenv

load_dotenv(override=True)

client = OpenAI(
    base_url="https://llm-api.amd.com/Unified/v1",
    api_key=os.environ["LLM_GATEWAY_KEY"],
    default_headers={"Ocp-Apim-Subscription-Key": os.environ["LLM_GATEWAY_KEY"]},
)

response = client.chat.completions.create(
    model="Claude-Opus-4.8",
    messages=[{"role": "user", "content": "Hello!"}],
)
```

### Open-source image-processing model
You can use any open-source image-processing AI models like (U-net, YOLO, ResNet, ...) that are available online.

## Assumptions & Open Questions
- Coordinate units are **micrometers (µm)**; no explicit unit field exists in the coordinate file.
- Row 2 of the coordinate Excel sheet is always blank and must be skipped when reading.
- The coordinate file uses a **top-down view** (reference corner = top-left), while the dye-and-pry images use a **bottom view** (reference corner = top-right). The required spatial transform is a horizontal mirror: `x_bv = -X_coord`, `y_bv = Y_coord`.
- ✅ **Confirmed (notebook 01):** 2 077 balls, 63 × 77 grid, 800 µm pitch, array extents 34 900 × 42 400 µm.
- ✅ **Confirmed scale (notebook 02):** affine transform per image gives ~0.203–0.222 px/µm (X) and ~0.198–0.219 px/µm (Y); rotation < 1°; ball pitch ~162–164 px. Each image has a slightly different scale due to varying stage position, so a separate affine is computed per image in the end-to-end pipeline.
- **No ground-truth labels exist yet.** Numerical accuracy validation will only be possible after the tool is built and manual labelling is performed in a separate step.
- **Category thresholds** (crack area as a percentage of total ball area):
  - Category A: exactly 0%
  - Category B: > 0% and ≤ 25%
  - Category C: > 25% and ≤ 50%
  - Category D: > 50% and ≤ 75%
  - Category E: > 75%
  - **When measurement uncertainty places a ball on a boundary, assign the more severe category** (e.g. if unsure between B and C, assign C).

## Steps
### Step 1: Research

- [x] **1.1 IPC-TM-650-2.4.53 review**
  Read `docs/ipc-tm-650-2.4.53 Dye and Pry.pdf` to understand:
  - What constitutes a crack versus normal dye penetration.
  - Visual features that distinguish categories A–E.
  - Any standard measurement methodology for crack area percentage.

- [x] **1.2 Coordinate transform verification**
  - 5 interior balls (Y9, C25, R15, G40, P20) were predicted using the mirror transform
    (`x_bv = -X_coord`, `y_bv = Y_coord`) with a linear scale derived from the array extents.
  - Hough circle detection placed all 5 detected centres within 41–75 px of the predicted positions,
    confirming the horizontal-mirror direction is correct.
  - The residual error is expected at this stage because the linear scale uses the full array extent
    as a proxy for the image field-of-view; the exact affine calibration (including margins) will be
    computed in `02_image_registration.ipynb` using RANSAC over all ~900+ detected balls.
  - **Key finding:** ball A10 is at Y = 21 200 µm (the maximum Y), which maps to pixel row ≈ 0
    (top edge of image) — edge balls may be partially clipped and should be handled as boundary cases.

- [x] **1.3 Sample image characterisation**
  - **Image sizes:** 7 925–8 554 × 9 295–9 841 px (vary slightly between images; 20× magnification).
  - **Ball size:** median radius ≈ 103–123 px across images → typical diameter **~210–220 px**.
  - **Ball pitch:** 800 µm physical → **~164 px** at the estimated 0.205 px/µm scale.
  - **Brightness:** dark background (global mean 112–129, std 67–77) with good dye/ball contrast.
  - **Detected ball count per image:** 887–1 102 (out of 2 077 total; edge balls and
    overlapping/fused balls account for the remainder — to be handled in registration notebook).
  - **Margins:** approximately 350–425 px (left/top) and 275–345 px (right/bottom) between the
    image border and the outermost ball row/column.
  - **No fiducial marks** were observed; alignment will rely on detected ball positions.
  - **ROI crop size for segmentation:** 256 × 256 px recommended (comfortably contains a single ball
    at ~220 px diameter with a small border).

- [ ] **1.4 Manual labelling of ~20 balls**
  - Select balls spanning different crack severities from at least one image.
  - Record the ball ID, estimated crack percentage, and assigned category.
  - Use these as a qualitative reference during classifier development.

- [x] **1.5 Literature review (summary)**
  - **Crack segmentation:** U-Net with encoder-decoder architecture and skip connections outperforms YOLO and ResNet for pixel-level crack segmentation (dice score ≈ 0.66 on benchmark datasets).
  - **Ball detection / image registration:** OpenCV SIFT keypoint matching + FLANN + RANSAC homography is standard for sub-pixel alignment of PCB images.
  - **BGA defect detection:** YOLO-APGC achieves 99.2% accuracy on BGA surface defect classification tasks.

### Step 2: Plan

- [x] **2.1 Pipeline overview**
  ```
  coordinates.xlsx
    └─ parse + mirror transform → bottom-view coords (µm)
         └─ image registration (SIFT + RANSAC)
              └─ µm → pixel transform per image
                   └─ ROI extraction per ball (~200 px diameter crop)
                        └─ crack segmentation (U-Net)
                             └─ crack area fraction → category A–E
                                  └─ <package>_BGA_label_output.xlsx
  ```

- [x] **2.2 Coordinate transform**
  ```python
  # From top-down view (coordinate file) to bottom-view (image space)
  x_bv = -X_coord   # horizontal mirror
  y_bv =  Y_coord   # no change
  ```
  Scale from µm to pixels is determined during image registration (2.3).

- [x] **2.3 Image registration strategy**
  1. Detect solder ball centre positions in the full image using Hough circle transform or blob detection.
     - Hough parameters (validated in Step 1.2): `dp=1.5`, `param1=60`, `param2=25`, `minRadius=60`, `maxRadius=130` (full-res); detect on 4× downsampled image for speed then scale back.
     - Typical yield: **887–1 102** circles detected per image (out of 2 077; edge/fused balls account for the rest).
  2. Match detected centres to the transformed coordinate grid using nearest-neighbour assignment and RANSAC to reject outliers.
  3. Compute an affine transform mapping µm coordinates to pixel coordinates — implemented as a **3-round pipeline** (validated in notebook 02):
     - Round 1: rough linear scale → approximate pixel positions → NN match → RANSAC.
     - Round 2: project all coords through M1 → tight NN re-match → RANSAC.
     - Round 3 (refinement): for each coarse inlier, crop a 140×140 px window (tight enough to isolate one ball at ~164 px pitch), Otsu-threshold, find the largest blob centroid (subpixel accuracy) → final RANSAC affine.
  4. Validate: RMSE of predicted centres vs. detected centres < 5 px (MVP); target < 2 px for production.
     - ✅ **Validated (notebook 02):** all 5 FP11_BLTC images pass. RMSE range 1.70–4.44 px, mean 3.38 px.
  5. Fallback: if automatic detection fails, provide a notebook cell for manual corner annotation.

- [x] **2.4 Crack segmentation approach**
  - Crop a fixed-size ROI of **256 × 256 px** centred on each ball's pixel coordinate (confirmed in Step 1.3: typical ball diameter ~210–220 px, so 256 px gives a comfortable border).
  - Run each ROI through a pre-trained U-Net (e.g. `segmentation-models-pytorch` with an ImageNet-pretrained encoder).
  - Post-process the binary mask with morphological operations (opening/closing) to remove noise.
  - Crack fraction = (crack pixels inside ball circle) / (ball interior pixels).
  - Map fraction to category using thresholds from the Assumptions section.

- [x] **2.5 LLM assistance (optional, later)**
  - The AMD gateway (`Claude-Opus-4.8` via OpenAI-compatible API) may be used for edge-case validation: pass a base64-encoded ball crop with a prompt asking for crack severity, and use the response as a second opinion when the segmentation confidence is low.
  - The LLM is **not** the primary classifier; it is an optional post-processing step.
  - Use the LLM setup code provided in the **Available resources** section above.

- [x] **2.6 Technology stack**
  ```
  python             ≥ 3.11
  uv                 environment management
  opencv-python      image processing (Hough circles, RANSAC, morphology, affine)
  numpy              array operations
  scipy              spatial indexing (cKDTree for nearest-neighbour matching)
  pandas             coordinate data handling
  openpyxl           read/write xlsx
  torch              deep learning backend
  segmentation-models-pytorch  pre-trained U-Net encoder-decoders
  openai             AMD LLM gateway (optional)
  python-dotenv      .env key loading
  jupyterlab         notebook environment
  matplotlib         visualisation inside notebooks
  ```

- [x] **2.7 Output file specification**
  - Path: `database/<package>/output/<package>_BGA_label_output.xlsx`
  - Columns: `BGA Number`, `X Coord`, `Y Coord`, then one column per image (named after the image filename, without extension) containing the category letter (A–E) for every ball.
  - Row order: same as the source coordinate file.

### Step 3: Build

All development at this stage is **notebook-only**. Each notebook is self-contained and can be run independently. Helper functions may be defined inside notebook cells or in small `.py` files in `src/` that the notebooks import.

**Environment setup** ✅ *(completed — `pyproject.toml`, `uv.lock`, and `.venv/` already present)*
```bash
uv init
uv add opencv-python numpy pandas openpyxl scipy torch segmentation-models-pytorch \
        openai python-dotenv jupyterlab matplotlib
```
Populate `LLM_GATEWAY_KEY` in the existing `.env` file at the project root before running any LLM-related cells.

**Notebooks in `src/notebooks/`:**

- [x] **3.1 `01_coordinate_exploration.ipynb`** *(all sanity checks passed)*
  Load and inspect the coordinate xlsx. Apply the mirror transform. Visualise the ball grid on a scatter plot. Confirm expected ball count (2 077).
  - **Results:** 2 077 balls ✅ · 63 × 77 grid ✅ · 800 µm pitch ✅ · 0 missing values ✅ · X symmetry ✅
  - Grid scatter plots (top-down and bottom-view orientations) generated and verified visually.

- [x] **3.2 `02_image_registration.ipynb`** *(all 5 images PASS)*
  Load one image. Detect ball centres (Hough circles). Match to transformed coordinates. Compute and display the affine transform. Report RMSE.
  - **Results:** RMSE 1.70–4.44 px (mean 3.38 px) across all 5 images ✅
  - Scale ~0.203–0.222 px/µm; rotation < 1°; pitch ~162–164 px.
  - 3-round pipeline: coarse Hough → coarse RANSAC affine → centroid blob refinement → final RANSAC.

- [ ] **3.3 `03_roi_extraction.ipynb`**
  Using the registration from notebook 02, crop ROIs for a sample of balls. Display a grid of crops to visually verify alignment.

- [ ] **3.4 `04_segmentation_prototype.ipynb`**
  Load a pre-trained U-Net. Run inference on sample ROIs. Display the binary crack mask overlay on each crop. Compute crack fraction.

- [ ] **3.5 `05_classification.ipynb`**
  Apply the category thresholds to crack fractions. Display the distribution of categories across one image.

- [ ] **3.6 `06_end_to_end_demo.ipynb`**
  Run the full pipeline (notebooks 01–05 steps combined) on all 5 images. Write the output xlsx. Display a summary table and annotated image overlay.

### Step 4: Test

- [x] **4.1 Coordinate sanity checks (notebook 01)** *(all passed)*
  - Ball count: **2 077** ✅
  - X range: −17 450 to +17 450 µm (symmetric) ✅
  - Grid: 63 unique X × 77 unique Y positions, pitch 800 µm (X and Y) ✅
  - Missing values: **0** ✅
  - Scatter plots confirmed correct grid pattern and mirror orientation ✅

- [x] **4.2 Registration quality check (notebook 02)** *(all passed)*
  - RMSE: 3.98, 3.06, 1.70, 3.71, 4.44 px across the 5 images — all < 5 px gate ✅
  - All 5 interior spot-check balls (Y9, C25, R15, G40, P20) mapped within image bounds ✅
  - Note: Hough detects 709–937 circles per image (not 2 077); edge and partially-fused balls are clipped — handled as boundary cases in ROI extraction (notebook 03).

- [ ] **4.3 ROI alignment spot-check (notebook 03)**
  - Visually inspect a random sample of 20 crops and confirm each is centred on a solder ball.
  - Check boundary crops (balls near the image edge) for partial occlusion.

- [ ] **4.4 Segmentation visual validation (notebook 04)**
  - Display segmentation masks for the ~20 manually labelled balls from Step 1.4.
  - Compare assigned categories to the manual labels as a qualitative (not numerical) check.
  - Note: numerical accuracy cannot be computed until ground-truth labels are collected in a separate labelling step after the tool is built.

- [ ] **4.5 End-to-end output check (notebook 06)**
  - Verify the output xlsx is created at the correct path.
  - Verify it has exactly 2 077 rows and 5 image columns (for the FP11_BLTC package).
  - Verify no `NaN` or missing values in any category column.
  - Verify all values are one of: A, B, C, D, E.

**4.6 Open validation items (to be addressed after ground-truth labelling)**
- Quantitative accuracy (per-category precision/recall) requires manually labelled reference data.
- U-Net may need fine-tuning on actual dye-and-pry images if zero-shot performance is visually poor.
- Registration and segmentation robustness across packages other than FP11 is unvalidated.

