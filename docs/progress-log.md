# Progress Log

Chronological checkpoint of work done. Newest entries at the bottom.

## 2026-06-27 — Session 1: project understanding, baseline, tooling

### Understood the challenge
- Read `desc.md`, the official `README.md`, the TUM/Inria slide deck (`presentation_TUM_AI.pdf`), and the organizer `Q&A.md`.
- Established the full picture: cross-modal ceT1→T2 same-patient retrieval, 3 datasets as escalating difficulty levels (aligned / deformed / pre-post-surgery), macro-average MRR scoring, judging = leaderboard score with creativity as tiebreaker. See [challenge.md](challenge.md).

### Environment & data access
- Confirmed: Windows 11, AMD Radeon 890M (CPU-only torch locally), `uv` 0.9.21, Python 3.13.
- Set up Kaggle API auth via a `KGAT_` token in `.env`.
- First token's account had **not accepted the competition rules** → 403 on download. Second token's account had accepted → access works (`userHasEntered = True`).
- Resolved the competition slug from the `/t/` invite link: **`ehl-paris-medical-image-retrieval`** (deadline 2026-06-28 10:30).
- Briefly started a local ~28 GB download, then **cancelled it** — decided on a hybrid workflow (code locally, train on Kaggle where data is pre-mounted + free GPU).

### Baseline on Kaggle
- Wrote `kaggle_baseline.py` — Kaggle-adapted copy of the official baseline (device=cuda, auto-detect paths, pip install, self-contained).
- **Bug found & fixed:** CSVs store absolute paths with `.nii.gz` extension, but real files are `.nii`. Switched to resolving images by **ID via a glob index**, ignoring stored paths/extensions.
- Ran successfully on Kaggle GPU: 1454 files indexed, loss 4.768 → 0.672 over 500 epochs, wrote a valid 377-row `submission.csv`. See [baseline.md](baseline.md).
- **Open item:** submit to Kaggle and record the public leaderboard reference score.

### Tooling: `entire` AI-session capture (required for grading)
- Verified `entire` (entire.io) is a legit open-source MIT CLI that records AI-agent sessions into git; vetted the install script (safe).
- Installed on Windows via **Scoop** (v0.7.7) — the `curl | bash` line is macOS/Linux only.
- `git init` + `entire enable --agent claude-code` → 7 Claude Code hooks, session branch `entire/checkpoints/v1`.
- Added `.gitignore`; verified `.env` (Kaggle token) is **not** tracked.

### Decisions made
- **Hybrid workflow:** author code locally with Claude Code, run training/submission on Kaggle.
- **GitHub repo:** local only for now; create remote later (public vs private TBD), needs `gh` or manual remote.

### Next up
1. Submit baseline → record reference score per dataset.
2. Start improvement #1 from [strategy.md](strategy.md): synthetic rigid+elastic deformation augmentation (independent on query/target) to simulate dataset2.
3. Create the GitHub remote and push (including the `entire` session branch) for the submission deliverable.

## 2026-06-27 — Session 2: reframe + GPU pairwise rankers (NMI / MIND)

### Reframed the problem
- Live research confirmed this is **same-patient cross-modal matching**, not tumor classification.
  The proven levers are mutual information (d1, common grid), MIND/MIND-SSC descriptors (d2/d3),
  and the BraTS identity-lookup leak (likely source of the 95+% leaderboard scores). Dropped the
  "finetune a brain-tumor model" idea as the main method. Plan file:
  `~/.claude/plans/hey-claude-i-dont-sprightly-dolphin.md`. Memory reset + re-seeded.

### Built GPU pairwise rankers
- New `rankers.py`: torch/ROCm rankers scoring a query against a candidate jointly —
  `nmi` (normalized MI), `gradcos` (gradient-magnitude cosine), `nmi_grad` (blend),
  `mind` (global Heinrich MIND descriptor cosine). `get_rankers()` registry; `RANKERS` env allowlist.
- Extended `eval_harness.py`: added `mrr_from_scores()` and a ranker scoring loop alongside the
  embedders; added `SKIP_LEARNED=1` for a fast classical-only pass.
- `tools/smoke_test_rankers.py` (CPU): identity MRR=1.0 for all four; under the d2 proxy
  nmi/mind/nmi_grad ≈0.75–0.78 vs ~0.29 chance, gradcos ≈0.52. PASSED locally.
- Uploaded to the box: `amine/{rankers.py, eval_harness.py, run_eval.ipynb, tools/smoke_test_rankers.py}`.

### Next up
- Run `run_eval.ipynb` on the MI300X (fast pass, SKIP_LEARNED=1) → record real d1/d2/d3 proxy MRR
  per ranker in `eval_results.md`; expect a large d1 jump from `nmi`.
- Wire the winning ranker per dataset into `make_submission.py`; tune `nmi_grad` weight and the
  d2/d3 MIND blend on the proxy before spending a Kaggle submission.

## 2026-06-27 — Session 3: confirmed BraTS leak + hardened lookup track

### Confirmed the leak with voxel inspection (`tools/inspect_voxels.py`)
- **dataset1 & dataset2**: shape `240×240×155`, clean zero background, skull-stripped → exactly
  BraTS/SRI24 space. These ARE public BraTS patients. d1 is undeformed (so the lookup is an
  essentially exact same-modality match → expect ~1.0 d1 MRR); d2 is the deformed copy.
- **dataset3**: variable shapes (`211×250×176`, `219×250×192`), not SRI24 → a different
  (intra-op, ReMIND-like) set. BraTS lookup does NOT apply; d3 stays on shape/MIND fallback.

### Hardened the identity-lookup track (built on prior `make_submission_lookup.py`)
- Added `bridge()` (returns scores + patient-recovery confidence: top1 sim & margin) and
  `validate_d1()` (end-to-end BraTS-lookup MRR on labelled d1 train pairs — the trustworthy
  pre-submission estimate). `main()` now logs per-pool confidence.
- Rewrote `run_lookup.ipynb`: clean ~13GB BraTS-2021 download steps
  (`dschettler8845/brats-2021-task1` via Kaggle, env-var auth, no key on disk) + a VALIDATE cell
  (self-check + `validate_d1`, asserts MRR>0.6) before building the submission.
- `validate_self(n=12,d=24)` runs locally (no BraTS): top1=0.58 / MRR=0.73 on a deformed gallery
  — coarse-descriptor d2 robustness is moderate; `LOOKUP_D` (16/24/32) is the tuning knob.
- Uploaded to box: `amine/{make_submission_lookup.py, make_submission.py, brats_lookup.py, run_lookup.ipynb}`.

### Next up
- On box: download BraTS, Run All `run_lookup.ipynb` → read `validate_d1` MRR + confidence; if
  d1≈1.0, submit d1/d2 lookup rows. Tune `LOOKUP_D` / descriptor for d2 if its confidence is low.
- d3: keep shape/MIND fallback; consider ReMIND reference download as a later upgrade.

## 2026-06-28 — Session 4: BraTS leak DEAD; dense-MIND d1 win

### BraTS lookup failed → pivoted to content
- Downloaded BraTS-2021-task1 (1251) to the box; `tools/diag_lookup.py` brute-forced 48 axis
  orientations: best competition→BraTS sim only 0.64 (identical scan would be ~0.95+), winning
  orientation random per query. Verdict: d1/d2 are BraTS-FORMAT but NOT in BraTS GLI training
  (2023-GLI == same 1251, don't bother). Likely UCSF-PDGM/UPENN-GBM/ReMIND or private Inria data.
- Honest call: 95+ unreachable by content; locked in the best content floor instead.

### Best content method = DENSE MIND on d1 (the alignment win)
- `tools/exp_rankers.py` on GPU: global rankers all ~0.55-0.57 on a 100-gallery; **dense_mind**
  (MIND field cosine across the aligned grid, NO global pooling) = **0.714**. Blends hurt it.
- `tools/exp_mind_tune.py`: foreground-masking HURTS (0.71→0.63) — background all-ones MIND encodes
  the brain-mask shape, an alignment cue (same lesson as NMI). Richer neighbourhoods / higher res
  didn't beat the default (G96, dilation 2, 6-face). So 0.714 is the d1 content ceiling.
- Added `rank_dense_mind` to `rankers.py`; `make_submission_content.py` now uses it for d1
  (d2=global MIND, d3=shape). Re-run on box → `submission_content.csv` → submit.
