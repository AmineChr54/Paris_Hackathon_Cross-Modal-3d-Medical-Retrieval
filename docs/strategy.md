# Strategy: beating the baseline

## Why the baseline fails on datasets 2 & 3

| Weakness | Consequence |
|----------|-------------|
| Trains **only on aligned dataset1** with just Gaussian-noise augmentation | No exposure to deformation (d2) or structural change (d3) → collapses there |
| **3 fixed-depth axial slices** (0.35/0.50/0.65 of z-range) | Implicitly assumes alignment; on d2/d3 the same slice index shows different anatomy. Discards most of the 3D volume |
| **Separate query/target encoders**, tiny CNN, no pretraining | Weak features, easy to overfit 350 pairs |

Because the score is `(d1 + d2 + d3)/3`, points on d2/d3 are where the competition is won.

## Roadmap (highest leverage first)

1. **Synthetic deformation augmentation (simulate Level 2).**
   During training, apply **independent** random rigid (rotation/translation) + non-linear elastic deformations to query and target. This teaches deformation-invariant, content-based matching directly from the aligned dataset1 pairs. Highest-leverage single change. (MONAI: `RandAffined`, `Rand3DElasticd`, `RandGridDistortiond`.)

2. **Use more of the volume / stop relying on alignment.**
   Move beyond 3 fixed axial slices: multi-plane (axial+coronal+sagittal) sampling, more slices, or a true **3D encoder**. Sample slices by anatomy/content rather than fixed z-fraction.

3. **Stronger encoders / pretraining.**
   Replace the tiny CNN with a stronger 2D backbone (ImageNet-pretrained) on multi-view slices, or a 3D medical SSL backbone. Consider a **shared** encoder + modality token instead of two separate encoders, to tighten the shared space.

4. **Robustness for Level 3 (pre/post-surgery + domain shift).**
   Intensity/histogram standardization, aggressive intensity & bias-field augmentation, and possibly an explicit registration or coarse alignment step. Local structure is unreliable here — favor global/anatomical descriptors.

5. **Retrieval-time tricks.**
   Test-time augmentation (average embeddings over several augmented views), embedding whitening / re-ranking, and query-gallery symmetric scoring.

## Validation discipline

- Public LB = validation rows only; we have local val labels? No — correct matches are hidden. So tune against the **public leaderboard MRR per dataset** (it's broken out by the macro-average structure), but respect the **100 submissions/day** cap.
- Use dataset1 (which we know is registered) for fast offline sanity checks of the training loop before spending submissions.

## Creativity angle (tiebreaker)

Judges use creativity to break near-ties, and deliverables include a pitch + live demo. An **interpretable** retrieval (e.g. showing *why* two cases match — attention maps, matched regions) adds value beyond the raw score and strengthens the demo.
