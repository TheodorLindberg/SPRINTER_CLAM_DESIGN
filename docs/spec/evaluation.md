# Spec · Evaluation (BEAM generation)

Contracts for [Stage 5](../design/07-evaluation.md) — how a [BEAM file](../formats/beam.md) is produced. [Overview](../design/07-evaluation.md) · **Specification** · [Implementation](../impl/evaluation.md).

## Unit & naming

One BEAM per **(biopsy, run)**: `{biopsy_id}__{run_id}.beam.h5`. A *run* is a trained model with one checkpoint per fold; it was trained on exactly one bundle, i.e. one `(cohort, stain, source_variant, embedding_model, patch_config)`. So a BEAM corresponds to that biopsy's scan **for the run's stain** — the run's stain and the bundle's stain must match (hard error otherwise).

## Checkpoint routing (the crux)

Which checkpoint predicts a given patient depends on the `subset`:

| `subset` | Rule |
|---|---|
| `development` | Each patient is predicted by the checkpoint of the **fold where it was in `test`** (out-of-fold). No patient is ever scored by a checkpoint that trained on it. |
| `holdout` | Holdout patients were in no fold → predicted by **all fold checkpoints, aggregated** (default: mean prediction). Record which checkpoints contributed. |
| `all` | A single model trained on everything → one checkpoint. |

This routing is what produces an honest CV-test score vs. a holdout score.

## What a BEAM contains (generation contract)

| Field | How it is produced |
|---|---|
| `prediction` | Bag-level model output (regression value, or class probabilities). For `holdout`, the aggregate across fold checkpoints. |
| `attention/{raw,sigmoid,rank}` | **Only for attention-based architectures** (CLAM). `raw` = model attention; `sigmoid` = squashed; `rank` = percentile. Mean-pool / regression heads have **no** `/attention` (omitted, not faked). |
| `patches/coords` | From the bundle's embedding file — **raw WSI frame**. |
| `outline/` (+ quartiles) | The scan's outline + biopsy-axis quartiles from [Stage 2](wsi-transformation.md). |
| `labels/` | True label from the bundle's `labels.csv`, joined on `(dataset_id, patient_id, biopsy_id)`; absent for label-free bundles. |
| root attrs | `run_id`, `model_experiment`, `checkpoints_used`, `bundle_id`, `cohort_id`, `subset`, `embedding_model_id`, `source_variant`, `patch_config_id`, `stain`, `evaluation_tag`, `biopsy_id/patient_id/dataset_id`, `membership_hash`, `git_commit` + forwarded scan metadata. |

## Aggregation (per-bag → per-biopsy)

A whole-scan bag is one `(biopsy, stain)` → one prediction → one BEAM (**1:1**; the "per-scan → per-biopsy" step is then just collection). If an experiment uses **per-quartile** or **multi-stain** bags, the per-bag results are aggregated into the biopsy's BEAM (predictions combined per the architecture; attention concatenated, tagged by quartile/stain). See [open issue: multi-stain bundles](../design/09-open-questions.md).

## Invariants

- Run stain == bundle stain.
- BEAM `patches/coords` equal the bundle embedding coords (same raw frame, same order).
- `development`: each patient's prediction uses **exactly** its out-of-fold checkpoint.
- `/attention` exists **iff** the architecture is attention-based.

## Acceptance criteria

- A `development` evaluation yields one BEAM per development biopsy, each from its out-of-fold checkpoint, and the predictions reproduce the run's recorded CV-test metrics.
- A `holdout` evaluation yields one BEAM per holdout biopsy, none of whose contributing checkpoints saw that patient (cross-check against fold assignments + `membership_hash`).
- Given fixed checkpoints, regeneration is deterministic.
