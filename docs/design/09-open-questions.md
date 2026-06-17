# Open Questions & Decisions

Working notes for the team. Most of the early design questions are now settled — those are recorded under [Resolved decisions](#resolved-decisions) below, each with a pointer to the fuller rationale in the [Appendix](12-appendix.md). The few that are still genuinely undecided are kept at the top.

---

## Still open

### Holdout ensembling policy

Holdout patients are scored by **all** fold checkpoints. The current default is the **mean** prediction; we still need to confirm this against the alternatives — median, best-fold, or keeping each per-checkpoint prediction separately.

### Combining input sources in CLAM training preparation

How to merge different input data sources when assembling CLAM training inputs is undecided.

### Exact bundle schema

The bundle manifest schema — shared by training and evaluation — must be pinned down before implementation. It is deferred on purpose while the surrounding stage contracts settle.

---

## Resolved decisions

A brief record of what's been decided and why. The reasoning behind each is in the [Appendix](12-appendix.md).

### Embedding reuse strategy {#embedding-reuse-strategy}

**Resolved: a file-level cache.** Embeddings are stored one HDF5 per `(scan, source variant, embedding model, patch config)`, with the configuration encoded in the file path, so the Snakemake DAG tracks reuse directly: an already-embedded configuration is skipped, a changed one writes a new file, and because the cohort is not in the path, a scan embedded once is reused by every cohort and bundle. This keeps reuse visible to the workflow with no separate store and no per-row bookkeeping.

If a future experiment plan sweeps overlap or patch size heavily on the same scans, the cheapest robust extension is **embed once at the finest grid and decimate** for coarser grids (a pure index operation) — adopted only if that need actually appears.

### Augmentation {#augmentation}

**Resolved: no augmentation in the first version.** Because foundation-model embeddings are largely invariant to the usual histology transforms (flips, rotations, stain/colour jitter), augmenting pixels and re-embedding yields little signal for substantial extra storage and bookkeeping. If it is wanted later, it belongs in **preprocessing** (the transforms change the pixels the embedding model sees, so each augmented patch must run through the foundation model), embedded and stored alongside the unaugmented set, with training choosing whether to sample it — added only once a held-out uplift is demonstrated.

### Holdout leakage {#patient-exclusion-leakage}

**Resolved: no fitted statistics in bundles, and holdout filtered out of every fold.** Bundles carry raw labels and embeddings only; anything *fitted* (label normalization, distribution-derived thresholds, class weights) is computed at training time from the training fold alone. With holdout patients kept out of every fold, holdout shares no derived state with development by construction. See [Cohorts, roles, and splits](02-data-model.md#cohorts-roles-and-splits).

*One thing left to verify:* that any embedding-time stain normalization is a fixed pretrained transform, not something fitted on this cohort — if it were fitted, it would need the same treatment.

### Metadata scope {#metadata-file-scope}

**Resolved: nested `metadata:` per level in the scan manifest**, forwarded into the bundle and BEAM grouped by level. The manifest is hierarchical (dataset → patient → biopsy → scan), so each value sits at the level it describes, is written once, and keeps its level tag all the way to the [BEAM](../formats/beam.md) `/metadata`. No separate metadata store. See [Data Ingestion · Scan manifest](03-data-ingestion.md#the-scan-manifest).

### Stains per bundle {#stains-per-bundle}

**Resolved: one stain per bundle (one model per stain) for now.** A biopsy's HE / Ki67 / PSA scans become separate bundles, each feeding its own model, so evaluation's per-bag → per-biopsy step stays 1:1.

A future **multi-stain, multi-target** model — for example, one model that takes both Ki67 and PSA embeddings for a biopsy and predicts both scores — is designed to *extend* these contracts rather than change them (single-stain stays the one-element case):

- **Bundle** — allow `stains: [Ki67, PSA]`; the bundle then carries one bag per `(biopsy, stain)`, already distinguishable via the stain field of `bag_id`.
- **Architecture** — one MIL branch per stain, each pooling its own bag, fused at a head with per-target outputs. Because each stain is pooled independently, no spatial alignment is needed and raw-frame training still holds.
- **Config & evaluation** — `target` becomes a set; balancing, metrics, attention, and predictions are all tracked per target, which turns the per-bag → per-biopsy step into a real aggregation.
