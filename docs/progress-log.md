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
