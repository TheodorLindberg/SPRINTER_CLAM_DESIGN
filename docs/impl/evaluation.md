# Impl · Evaluation (BEAM generation)

Implementation notes for [Stage 5](../design/07-evaluation.md). [Overview](../design/07-evaluation.md) · [Specification](../spec/evaluation.md) · **Implementation**.

Reference libraries: **PyTorch** (inference), **h5py** (BEAM), **numpy/pandas**.

> These are design-level notes, not a prescriptive recipe. The authoritative behaviour is the [evaluation spec](../spec/evaluation.md) (routing, BEAM contract, invariants); a real implementation will be written against the actual codebase.

## Inference is organised per-model, repeated per sweep member

Load each checkpoint **once** and batch through it every patient it is responsible for — never load the model inside a per-patient loop ([why](../spec/evaluation.md#checkpoint-routing-the-crux)). The [checkpoint routing](../spec/evaluation.md#checkpoint-routing-the-crux) maps onto this directly, run **once per contributing model** in the sweep (each model — each `fold_seed × model_seed` combination of one `run_family` — has its own per-fold checkpoints):

- **`development`** — for each fold, load its checkpoint and score that fold's `test` patients (out-of-fold). Each patient is scored exactly once per model.
- **`holdout`** — for each fold checkpoint, score all holdout patients; accumulate per patient and aggregate at the end (default: mean), per model.
- **`all`** — a single full-data checkpoint scores everyone, per model.

Per-patient, per-model results are then collected into one BEAM per biopsy, with every contributing model's result kept distinct.

The same "load once, reuse" rule applies to the data, not just the model — and now extends across the sweep, not just across one model's fold checkpoints: every model in a sweep trains on the same bundle, so the bundle's embeddings are loaded **once** and reused across every checkpoint of every contributing model, never re-read per patient, per checkpoint, or per model (see [Bag I/O](training.md#bag-io-load-once-reuse-across-the-sweep)). Loading checkpoints once **and** embeddings once — for the whole sweep — is what keeps evaluation off the I/O floor.

## Prediction details

- **Aggregation, per model.** If a model's run used `target_normalization`, de-normalize each of its checkpoint's predictions with that checkpoint's own stored stats (back to label units) *before* combining. `holdout` then takes the mean across that model's fold checkpoints; `development` / `all` use that model's single contributing checkpoint as-is. This never blends across models — each model's combined prediction is its own.
- **Attention, per model.** Written only for a model whose architecture is attention-based, in three transforms: `raw` (model weights), `sigmoid = σ(raw)`, `rank = percentile(raw)`. A non-attention model omits `attention/` entirely for its group — never write zeros. A sweep may freely mix attention and non-attention models in the same BEAM.
- **Sweep invariants.** Every contributing model's run record must agree on `run_family`, `bundle_id`, `cohort_id`, `membership_hash`, and `subset` before combining — a mismatch means two different sweeps were accidentally mixed, and is a hard error.

## Writing the BEAM

One HDF5 file per `(biopsy, sweep)`, `{biopsy_id}__{run_family}.beam.h5`, holding: shared provenance attrs (sweep id, contributing model ids, subset, …), patch coordinates in the **raw frame** (same order as the embeddings, shared by every model), the scan outline with quartiles, and the true label when present — plus, under `models/{run_id}/` for each contributing model, that model's own prediction, attention transforms (if any), and a full copy of its `RunRecord` stats (architecture, hyperparameters, target_normalization, metrics, checkpoints used, git commit). HDF5 is appendable, so later enrichment steps (extra heatmaps, contributions) add datasets without rewriting. Exact field layout: [BEAM format](../formats/beam.md) and the [generation contract](../spec/evaluation.md#what-a-beam-contains-generation-contract).

## Notes

- Validate run stain == bundle stain before inference, for every contributing model.
- Holdout: record every contributing checkpoint id per model; cross-check none trained on the patient.
- Per-quartile / multi-stain bags: aggregate per biopsy per model (predictions combined per architecture; attention concatenated with quartile/stain tags) — the per-model fan-in into one BEAM is on top of this, not instead of it.
