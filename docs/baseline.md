# Baseline

The organizers provide `slice_clip_baseline.py` (a MONAI + PyTorch baseline). It is intentionally weak — its purpose is to demonstrate the data format, preprocessing, training loop, and submission generation, not to be a strong solution.

Our Kaggle-adapted copy is `kaggle_baseline.py` at the repo root.

## Architecture (the "tiny 2D-slice CLIP")

1. **Preprocessing (MONAI):** load NIfTI → channel-first → RAS orientation → resample to 1 mm → scale intensity to [0,1] → extract **3 axial slices** at depth fractions `0.35, 0.50, 0.65` of the occupied (non-empty) z-range → resize each to **96×96** → stack as a 3-channel image. Cached via `PersistentDataset`. Tiny Gaussian-noise augmentation during training only.
2. **Encoders:** two small 2D CNNs (a `query_encoder` and a `target_encoder`), 3 conv+pool blocks → MLP → 128-d embedding, L2-normalized.
3. **Loss:** CLIP-style symmetric in-batch contrastive (cross-entropy over the query↔target similarity matrix), fixed similarity scale 5.0.
4. **Ranking:** cosine similarity between query and gallery embeddings; argsort descending.
5. **Training data:** **only dataset1's 350 aligned pairs.**

Key config: `epochs=500, batch_size=128, lr=1e-3, embedding_dim=128, image_size=96`.

## Kaggle adaptations (`kaggle_baseline.py`)

Changes from the official script, all in the config/IO layer (model logic unchanged):

- **Device:** prefer `cuda` (Kaggle GPU), fall back to mps/cpu.
- **Paths:** auto-detect `DATA_ROOT` by globbing for `**/dataset1/train_pairs.csv` under `/kaggle/input`. Output to `/kaggle/working/submission.csv`, cache to `/kaggle/working/.monai_persistent`.
- **pip install** of `monai` + `nibabel` at the top (torch is preinstalled on Kaggle).
- **Self-contained:** hardcoded list of the 6 (query_csv, gallery_csv) prediction sets instead of argparse.

### The path-resolution bug & fix (important)

The competition CSVs store **absolute** image paths baked in from the organizers' environment (`/kaggle/input/competitions/.../<id>.nii.gz`), and the stored extension is **`.nii.gz`** while the actual files are **`.nii`**. The original `resolve()` returned absolute paths as-is, so it looked for non-existent `.nii.gz` files → `FileNotFoundError`.

**Fix:** ignore the stored paths entirely. Build a glob index of every `*.nii*` file under `DATA_ROOT` keyed by its **ID** (filename stem = the `query_id`/`target_id`), and resolve images by ID. This sidesteps both the absolute-path and extension mismatch and is robust to the exact mount location.

## Reference run results (Kaggle, GPU)

- `Detected DATA_ROOT: /kaggle/input/competitions/ehl-paris-medical-image-retrieval`
- `Indexed 1454 image files`, `device: cuda`, 700 train images, 754 inference images, 350 train pairs.
- Training converged: loss **4.768 → 0.672** over 500 epochs.
- First "Embedding (query)" pass took ~7 min (one-time MONAI cache build for the 754 inference volumes); "Embedding (target)" was instant (cache hit).
- Wrote **377 rows** to `submission.csv`.

**Public leaderboard score: _pending submission_** (update here once submitted).

### Expectation
This baseline should score non-trivially on **dataset1** (aligned) but near-random on **dataset2/3**, because it trains only on aligned data with trivial augmentation and relies on fixed-depth axial slices (which assume alignment). That gap is the target — see [strategy.md](strategy.md).
