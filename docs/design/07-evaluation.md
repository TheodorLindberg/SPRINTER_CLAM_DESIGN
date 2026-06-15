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

BEAM — *Biopsy Evaluation & Attention Map* — is the project's own per-biopsy, per-run result format: one appendable HDF5 file holding the prediction, per-patch attention (for attention models), patch coordinates, the tissue outline, true labels where available, and full provenance. HDF5 is used because it is appendable — later enrichment steps add datasets without breaking existing readers.

Its layout, datasets, and attributes are defined once in the **[BEAM format](../formats/beam.md)**; how each value is produced (checkpoint routing, aggregation, de-normalization) is the **[evaluation spec](../spec/evaluation.md)**.

---

## Reports

Aggregated across biopsies and models from the BEAM files — performance tables and per-model summaries. (Exact report contents TBD.)
