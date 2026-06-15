# Impl · Evaluation (BEAM generation)

Implementation notes for [Stage 5](../design/07-evaluation.md). [Overview](../design/07-evaluation.md) · [Specification](../spec/evaluation.md) · **Implementation**.

Reference libraries: **PyTorch** (inference), **h5py** (BEAM), **numpy/pandas**.

> These are design-level notes, not a prescriptive recipe. The authoritative behaviour is the [evaluation spec](../spec/evaluation.md) (routing, BEAM contract, invariants); a real implementation will be written against the actual codebase.

## Inference is organised per-model

Load each checkpoint **once** and batch through it every patient it is responsible for — never load the model inside a per-patient loop ([why](../spec/evaluation.md#checkpoint-routing-the-crux)). The [checkpoint routing](../spec/evaluation.md#checkpoint-routing-the-crux) maps onto this directly:

- **`development`** — for each fold, load its checkpoint and score that fold's `test` patients (out-of-fold). Each patient is scored exactly once.
- **`holdout`** — for each fold checkpoint, score all holdout patients; accumulate per patient and aggregate at the end (default: mean).
- **`all`** — a single full-data checkpoint scores everyone.

Per-patient results are then collected into one BEAM per biopsy.

The same "load once, reuse" rule applies to the data, not just the model: the bundle's embeddings are static, so load them once and reuse across all checkpoints rather than re-reading per patient or per checkpoint (see [Bag I/O](training.md#bag-io-load-once-reuse-across-the-sweep)). Loading checkpoints once **and** embeddings once is what keeps evaluation off the I/O floor.

## Prediction details

- **Aggregation.** If the run used `target_normalization`, de-normalize each checkpoint's prediction with that checkpoint's own stored stats (back to label units) *before* combining. `holdout` then takes the mean across fold checkpoints; `development` / `all` use the single contributing checkpoint as-is.
- **Attention.** Written only for attention-based architectures, in three transforms: `raw` (model weights), `sigmoid = σ(raw)`, `rank = percentile(raw)`. Non-attention architectures omit `/attention` entirely — never write zeros.

## Writing the BEAM

One HDF5 file per `(biopsy, run)`, `{biopsy_id}__{run_id}.beam.h5`, holding: provenance attrs (run, checkpoints used, subset, …), patch coordinates in the **raw frame** (same order as the embeddings), the prediction, the attention transforms (if any), the scan outline with quartiles, and the true label when present. HDF5 is appendable, so later enrichment steps (extra heatmaps, contributions) add datasets without rewriting. Exact field layout: [BEAM format](../formats/beam.md) and the [generation contract](../spec/evaluation.md#what-a-beam-contains-generation-contract).

## Notes

- Validate run stain == bundle stain before inference.
- Holdout: record every contributing checkpoint id; cross-check none trained on the patient.
- Per-quartile / multi-stain bags: aggregate per biopsy (predictions combined per architecture; attention concatenated with quartile/stain tags).
