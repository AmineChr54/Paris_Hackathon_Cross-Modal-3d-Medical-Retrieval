# Setup & Workflow

## Hardware / environment

| | |
|---|---|
| OS | Windows 11 |
| Local GPU | AMD Radeon 890M (integrated) — **no CUDA/MPS**, so local PyTorch is **CPU-only** |
| Tooling | `uv` 0.9.21, Python 3.13, Git 2.49, Scoop |
| Compute for training | **Kaggle notebooks** (free GPU: T4 ×2 or P100) |

> Possible future option: the team may get access to unlimited **AMD GPUs** for the hackathon, which would let us move training off Kaggle.

## The hybrid workflow (decided)

- **Code is authored locally** in this repo with Claude Code (so it's captured by `entire`, see below).
- **Training + submission run on Kaggle** — the competition data is pre-mounted there and the GPU is free.
- We paste `kaggle_baseline.py` (and future improved scripts) into a Kaggle GPU notebook and Run All.

Rationale: local hardware is CPU-only; Kaggle gives a free GPU and the ~28 GB dataset is already mounted (no download needed).

## Kaggle data access

- Competition slug: **`ehl-paris-medical-image-retrieval`**.
- Auth: a `KGAT_`-prefixed Kaggle API token stored in `.env` as `KAGGLE_API_TOKEN` (git-ignored). Used with the `kaggle` CLI ≥ 2.x.
- **You must accept the competition rules** at
  `https://www.kaggle.com/competitions/ehl-paris-medical-image-retrieval/rules`
  before downloading data or submitting (a 403 on download = rules not yet accepted).
- On Kaggle, data mounts at `/kaggle/input/competitions/ehl-paris-medical-image-retrieval/`.
- We chose **not** to download the ~28 GB dataset locally (hybrid workflow makes it unnecessary).

## `entire` CLI — AI-session capture (required for grading)

The organizers require [`entire`](https://github.com/entireio/cli) — an open-source (MIT) CLI that records AI-agent coding sessions into git history (on a separate `entire/checkpoints/v1` branch), so graders can see how the solution was built with AI. It ties into the "creativity" judging tiebreaker.

**Windows install is via Scoop** (NOT the `curl | bash` line, which is macOS/Linux only and would fetch the wrong binary):

```powershell
scoop bucket add entire https://github.com/entireio/scoop-bucket.git
scoop install entire/cli
```

Then, in the project (non-interactive):

```powershell
entire enable --agent claude-code --init-repo --no-github
```

This was done: entire v0.7.7 installed, repo initialized, 7 Claude Code hooks + a search subagent installed, `.entire/settings.json` written, and the `entire/checkpoints/v1` session branch created. `.env` is git-ignored so the Kaggle token is never committed.

### Still TODO for submission
- Create the **GitHub remote** and push `master` + the session branch (local repo only so far; no `gh` CLI installed yet). Decide public vs private per organizer requirements.
