# Spec · Model Training

Contracts for [Stage 4](../design/06-model-training.md). [Overview](../design/06-model-training.md) · **Specification** · [Implementation](../impl/training.md).

## Fold assignment (generated at train time)

Per `(seed_set, fold_seed)`, over the bundle's **development** patients only:

| Field | Type | Notes |
|---|---|---|
| `patient_id` | str | development patients only; holdout excluded |
| `fold` | int | `0 … n_folds-1` |
| `split` | enum | `train` / `val` / `test` (per the active fold) |

Stored alongside the run; **not** in the bundle.

## Run record (`run.json`, one per run)

```text
run_id, model_experiment, bundle_id, cohort_id
target, subset, seed_set, fold_seed, model_seed
architecture { family, type, params }
hyperparameters { ... }
metrics { train, val, test: {<metric>: value} }
checkpoints { fold_k: path }
membership_hash, git_commit, started_at, finished_at
```

All run records aggregate into **`runs.parquet`** — one row per run, columns = the flattened tags + headline metrics + paths. This is the report backbone and the export table.

## Metrics by label type

| Label type | Metrics |
|---|---|
| Regression | MAE, Spearman, R², Huber loss |
| Binary / classification | AUROC, accuracy, F1 (+ as appropriate) |

The architecture family fixes which apply: a `clam` / `non_clam` run is classification (binned target if the label is continuous) and emits the classification metrics; a `regression` run emits the regression metrics.

## Invariants

- No patient appears in more than one of `train` / `val` / `test` within a fold (patient-level).
- No `holdout` patient appears in any fold.
- Any fitted quantity (label mean/std, class weights, thresholds) is computed from the **train fold only**.
- `membership_hash` recorded in the run matches the cohort the `seed_set` was computed against; a mismatch is a hard error.

## Acceptance criteria

- A model experiment with *R* runs × *F* fold_seeds × *M* model_seeds produces `R·F·M` run records, each with a checkpoint per fold.
- Re-running with identical seeds reproduces fold assignments and (up to backend nondeterminism) metrics.
- `runs.parquet` round-trips: every run directory has exactly one row and vice versa.
