# Spec · Evaluation (BEAM generation)

Contracts for [Stage 5](../design/07-evaluation.md) — how a [BEAM file](../formats/beam.md) is produced. [Overview](../design/07-evaluation.md) · **Specification** · [Implementation](../impl/evaluation.md).

## Unit & naming

One BEAM per **(biopsy, sweep)**: `{biopsy_id}__{run_family}.beam.h5`. A *sweep* is every `fold_seed × model_seed` model fanned out from one experiment-config run entry (`run_family` — e.g. `"ki67_conch"`); each contributing model keeps its own prediction/attention/stats under `models/{run_id}/` inside the one file (the [BEAM format](../formats/beam.md) lays out the per-model layout). Every model in a sweep was trained on the **same** bundle, i.e. one `(cohort, stain, source_variant, embedding_model, patch_config)` — a BEAM corresponds to that biopsy's scan **for the sweep's stain**, and every contributing model's stain must match the bundle's stain (hard error otherwise).

## Checkpoint routing (the crux)

Routing is computed **per model** in the sweep — each model has its own per-fold checkpoints, so which checkpoint predicts a given patient still depends on the `subset` exactly as before, just evaluated once per contributing model:

| `subset` | Rule |
|---|---|
| `development` | Each patient is predicted by that model's checkpoint of the **fold where it was in `test`** (out-of-fold). No patient is ever scored by a checkpoint that trained on it. |
| `holdout` | Holdout patients were in no fold → predicted by **all of that model's fold checkpoints, aggregated** (default: mean prediction). Record which checkpoints contributed, per model. |
| `all` | A single model trained on everything → one checkpoint. |

This routing is what produces an honest CV-test score vs. a holdout score — for **every** model in the sweep independently.

!!! note "Inference is organised per-model, not per-patient"
    The routing above says *which* checkpoint scores *which* patient for *one* model; it does **not** imply scoring one patient at a time, nor does it imply combining models. Each checkpoint should be loaded **once** and all the patients it is responsible for batched through it (for `development`, the fold's test patients; for `holdout`, every patient) — and the same load-once-reuse discipline applies across the sweep's models, since they all share one bundle. Iterating patient-by-patient and reloading the model inside that loop is O(patients × checkpoints) disk loads and becomes badly I/O-bound — the failure mode to avoid. This is a performance guideline, not a contract: any order that respects the routing table and the invariants below is valid.

## What a BEAM contains (generation contract)

The BEAM **layout, datasets, and attributes** are defined once in the [BEAM format](../formats/beam.md). This spec fixes only **how the values are produced**. Fields shared across the whole sweep (one bag, one biopsy) are produced once; fields under each model's own `models/{run_id}/` group are produced **independently per model**:

| Field | Scope | How it is produced |
|---|---|---|
| `models/{run_id}/prediction` | per model | Bag-level model output (regression value, or class probabilities). For `holdout`, the aggregate across that model's fold checkpoints. |
| `models/{run_id}/attention/{raw,sigmoid,rank}` | per model | **Only for attention-based architectures** (CLAM). `raw` = model attention; `sigmoid` = squashed; `rank` = percentile. Mean-pool / regression heads have **no** `attention/` for that model (omitted, not faked) — a sweep may freely mix attention and non-attention models. |
| `models/{run_id}/@checkpoints_used`, plus `@fold_seed`, `@model_seed`, `@architecture_json`, `@hyperparameters_json`, `@target_normalization_json`, `@metrics_json`, `@git_commit` | per model | A full copy of that model's contributing `RunRecord` fields — the BEAM is self-describing per model without re-opening its `run.json`. |
| `patches/coords` | shared | From the bundle's embedding file — **raw WSI frame**, same order as the embeddings fed to every model (they all train/infer on the same bundle). |
| `outline/` (+ quartiles) | shared | The scan's outline + biopsy-axis quartiles from [Stage 2](wsi-transformation.md). |
| `labels/` | shared | True label from the bundle's `labels.csv`, joined on `(dataset_id, patient_id, biopsy_id)`; absent for label-free bundles. The true label does not vary by model. |
| root attrs | shared | Provenance per the format's [root attributes](../formats/beam.md#root-attributes); `model_ids` lists every contributing model (computed by the writer from what was actually written, never hand-supplied), and `membership_hash` / `git_commit` capture reproducibility state for the evaluation run itself. |

## Sweep invariants

Every contributing model's `RunRecord` in one BEAM call must agree on `run_family`, `bundle_id`, `cohort_id`, `membership_hash`, and `subset` — they are, by construction, every fold_seed/model_seed combination of **one** experiment-config run entry trained on **one** bundle. A mismatch (e.g. accidentally mixing two different sweeps) is a hard error.

## Aggregation (per-bag → per-biopsy → per-sweep)

A whole-scan bag is one `(biopsy, stain)` → one prediction per contributing model → one BEAM holding every model's contribution (**1:N within one file**, where N is the sweep's model count; the "per-scan → per-biopsy" step is collection, and "per-model" combination never blends across models — each keeps its own prediction/attention distinct). **One stain per bundle is the current rule**, so the per-model fan-in is exact. Per-quartile bags, or the future [multi-stain, multi-target models](../design/09-open-questions.md#stains-per-bundle), turn the per-model step into a deeper aggregation (predictions per target; attention tagged by quartile/stain).

## Invariants

- Every contributing model's run stain == bundle stain.
- BEAM `patches/coords` equal the bundle embedding coords (same raw frame, same order) — shared by every model.
- `development`: each patient's prediction, for each model, uses **exactly** that model's out-of-fold checkpoint.
- A model's `attention/` group exists **iff** that model's architecture is attention-based.
- Each model's `prediction` is in **label units** — de-normalized using that model's own per-checkpoint `target_normalization` when it was enabled; otherwise the model already outputs label units.
- See [Sweep invariants](#sweep-invariants) above for the cross-model consistency checks.

## Acceptance criteria

- A `development` evaluation yields one BEAM per development biopsy, holding every contributing model's prediction from its own out-of-fold checkpoint; each model's predictions reproduce that model's recorded CV-test metrics.
- A `holdout` evaluation yields one BEAM per holdout biopsy, where for every contributing model none of its contributing checkpoints saw that patient (cross-check against fold assignments + `membership_hash`).
- Given fixed checkpoints, regeneration is deterministic.
- Contributing models that disagree on a sweep invariant (e.g. trained on different bundles) are rejected, not silently merged.
