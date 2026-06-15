# Stage 5 · Evaluation

Once models are trained, this stage measures how good they actually are. It runs a trained model over the biopsies and, for each one, records the prediction it made and which patches it paid attention to, saving all of that as one result file per biopsy — a **BEAM** file. Those files are the raw material for both the performance reports here and the [heatmaps](08-heatmaps.md) in the next stage.

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

BEAM — *Biopsy Evaluation & Attention Map* — is the project's own per-biopsy, per-model result format, stored as HDF5 (`{biopsy_id}__{model_id}.beam.h5`). HDF5 is used because it is appendable: later enrichment steps can add fields without breaking readers that already exist.

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
