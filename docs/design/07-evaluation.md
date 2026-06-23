# Stage 5 · Evaluation

Once models are trained, this stage measures how good they actually are. It runs every model in a seed sweep over the biopsies and, for each biopsy, records each model's prediction and which patches it paid attention to, saving all of that as one result file per biopsy — a **BEAM** file holding every contributing model's prediction/attention/stats side by side. Those files are the raw material for both the performance reports here and the [heatmaps](08-heatmaps.md) in the next stage.

> **In** a bundle (chosen subset) + a trained seed sweep · **Out** one BEAM file per biopsy per sweep (every model's contribution inside), + reports

*Go deeper: [Specification](../spec/evaluation.md) · [Implementation](../impl/evaluation.md).*

```mermaid
flowchart LR
    INF[Inference\nper-scan H5, per model] --> AGG[Aggregate\nper-biopsy, per-sweep]
    AGG --> BEAM[BEAM file\n.beam.h5\nmodels/{run_id}/ per model]
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

One **BEAM file per biopsy per sweep** — every model in the sweep contributes its own prediction/attention/stats into that one file — plus aggregate reports.

---

## The BEAM format

BEAM — *Biopsy Evaluation & Attention Map* — is the project's own per-biopsy, per-sweep result format: one appendable HDF5 file holding, per contributing model, its prediction and per-patch attention (for attention models) under `models/{run_id}/`, plus the patch coordinates, tissue outline, true labels where available, and full provenance shared by every model in the sweep. HDF5 is used because it is appendable — later enrichment steps add datasets without breaking existing readers.

Its layout, datasets, and attributes are defined once in the **[BEAM format](../formats/beam.md)**; how each value is produced (checkpoint routing, aggregation, de-normalization) is the **[evaluation spec](../spec/evaluation.md)**.

---

## Reports

Aggregated across biopsies and models from the BEAM files — performance tables and per-model summaries. (Exact report contents TBD.)
