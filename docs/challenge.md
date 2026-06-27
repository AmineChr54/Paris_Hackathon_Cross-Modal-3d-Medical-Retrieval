# The Challenge

**Track:** Cross-modal Content-based Retrieval for 3D Medical Images
**Organizers:** Inria · Paris Brain Institute (ICM) · PRAIRIE Paris AI School
**Presenters:** Reuben Dorent (PI at Inria, junior fellow at PRAIRIE) and Nicolas Stellwag (MSc at TUM, intern at Inria) — presented 27 June 2026.

## Clinical motivation

The challenge is motivated by **brain-tumor neurosurgery**. Surgical resection is the critical first step for most brain tumors, balancing two competing goals:

- **Maximal tumor removal** → better overall survival
- **Minimal functional damage** → preserve quality of life

Medical data is **multimodal** (structural MRI: T1, ceT1, T2, FLAIR; functional MRI; intraoperative ultrasound; omics; histopathology). The clinical dream is **content-based retrieval**: when a new case arrives, automatically find similar previous cases by image appearance (not metadata), inheriting their known outcomes to guide the surgical plan.

The core obstacle is the **modality gap**: the same patient looks completely different across imaging protocols, so pixel comparison fails. A model must map both modalities into a **shared embedding space** where the same patient's scans land close together.

## The task

> Given a query 3D MRI (**contrast-enhanced T1 / ceT1**) and a gallery of target images (**T2**), rank the gallery so the T2 from the **same patient** is ranked as high as possible.

This is a bi-encoder retrieval problem (CLIP-style, but two MRI modalities instead of image+text).

- **Query modality:** T1 post-contrast (ceT1)
- **Target modality:** T2
- All volumes: 3D NIfTI, RAS orientation, 1.0×1.0×1.0 mm spacing. No intensity normalization / skull stripping / cropping applied. **Shapes can differ** between query and target, especially in datasets 2 and 3. (Note: actual files on Kaggle are `.nii`, not `.nii.gz`.)

## The three datasets = three difficulty levels

You only get **labelled training pairs from dataset1** (all perfectly aligned). You must generalize to 2 and 3 with **no labels** there. This is the heart of the challenge.

| Level | Dataset | What changes | Why it's hard |
|-------|---------|--------------|---------------|
| **1 — perfectly aligned** | dataset1 | ceT1 & T2 registered to a common grid, voxel-for-voxel | Easy; anatomy in the same location, only intensities differ |
| **2 — non-linear deformations** | dataset2 | Random rigid rotation/translation + non-linear deformation applied **independently** to query and target | Geometry-based shortcuts break; needs deformation-invariant features |
| **3 — before/after surgery** | dataset3 | Preop ceT1 vs intraop T2; tissue removed/shifted, parts missing, **plus a different hospital/scanner** | True domain + content shift; anatomy itself changed |

### Counts

```
dataset1:  train pairs 350 | val 40/40  | test 100/100
dataset2:  (no train)       | val 40/40  | test 100/100
dataset3:  (no train)       | val 20/20  | test  77/77
```

Total indexed image files on Kaggle: **1454**. Train images = 700 (350 pairs × 2). Inference images (all val+test queries+galleries) = 754.

## Retrieval pools

The three datasets are **independent** pools. Always rank a query only against the gallery from the **same dataset and same split**. Never mix datasets or val/test.

## Evaluation

- For each query, rank all gallery targets most→least similar. Reciprocal rank = `1 / rank` of the true match (absent/omitted → 0).
- **Per-dataset MRR**, then macro-averaged:

```
score = (dataset1_MRR + dataset2_MRR + dataset3_MRR) / 3
```

- **Public leaderboard** = validation query rows; **private leaderboard** = test query rows (final ranking).
- Because it's a macro-average over 3 datasets, **each dataset counts equally** regardless of size — a dataset3 query (only 77) is worth far more per-query than a dataset1 one. Excelling only on the easy dataset1 caps you at ~0.33.

### Judging (from organizer Q&A)

1. **Primary:** Macro-average ranking score (the Kaggle leaderboard).
2. **Tiebreaker:** if scores are statistically indistinguishable, **creativity** decides.
3. **Deliverables:** Pitch Deck + GitHub Repository + Live Demo.

## Submission format

One combined CSV (all datasets, val+test) — `377` query rows total + header:

```
query_id,target_id_ranking
q_...,g_... g_... g_... ...
```

Each row must contain a **full ranking of every target ID** from that query's same-dataset, same-split gallery, space-separated, best→worst. IDs are globally unique, so no dataset column is needed. Partial submissions allowed (omitted datasets score 0). **Limit: 100 submissions/team/day.**

Expected ranking lengths: d1 val 40 / d1 test 100 / d2 val 40 / d2 test 100 / d3 val 20 / d3 test 77.
