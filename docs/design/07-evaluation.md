# Stage 5 · Evaluation

Runs inference and writes one **BEAM** file per biopsy per **run** (a run = a trained model with per-fold checkpoints). Heatmaps are produced separately in [Stage 6](08-heatmaps.md).

> **In** a bundle (chosen subset) + a trained run · **Out** one BEAM file per biopsy per run, + reports

*Go deeper: [Specification](../spec/evaluation.md) · [Implementation](../impl/evaluation.md).*

```mermaid
flowchart LR
    INF[Inference\nper-scan H5] --> AGG[Aggregate\nper-biopsy, per-model]
    AGG --> BEAM[BEAM file\n.beam.h5]
    BEAM --> REP[Reports]
    BEAM --> HM[Stage 6 · Heatmaps]
```

---

## Evaluation step

### Input

- Path to bundle.
- **Subset** — typically `holdout` (the locked patients, scored once), or `development` for cross-validated test predictions.
- Path to model (checkpoints, folds, parameters/architecture).
- Evaluation tag — a readable name for the run.

!!! note "Two distinct scores"
    A **CV test score** (`development`, cross-validated) and a **holdout score** (`holdout`, the locked patients) are different numbers — keep them labeled as such. See [Cohorts, roles, and splits](02-data-model.md#cohorts-roles-and-splits).

!!! warning "Checkpoint routing"
    `development` patients are scored by their **out-of-fold** checkpoint (the fold where they were `test`); `holdout` patients are scored by **all** fold checkpoints, aggregated. This is the crux of BEAM generation — see the [spec](../spec/evaluation.md#checkpoint-routing-the-crux).

### Output

One **BEAM file per biopsy per model**, plus aggregate reports.

---

## The BEAM format

**BEAM** — *Biopsy Evaluation & Attention Map* — is the project's own per-biopsy, per-model result format, stored as **HDF5** (`{biopsy_id}__{model_id}.beam.h5`). HDF5 is chosen because it is **appendable**: enrichment steps can add fields without breaking existing readers.

Roughly, one BEAM file holds:

- **Attention** per patch — raw, sigmoid, rank.
- **Prediction(s)** for the biopsy and **true labels** where available.
- **Patch coordinates** (WSI frame) and patch size.
- **Tissue outline** used, as a polygon array, optionally divided into quartiles.
- **Provenance** — patient, stain, source variant, model, embedding model, patch config.
- **Free-form metadata**; quartile carried as metadata.

→ Full layout and field mapping: **[BEAM format spec](../formats/beam.md)**.

---

## Reports

Aggregated across biopsies and models from the BEAM files — performance tables and per-model summaries. (Exact report contents TBD.)
