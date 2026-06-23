# Spec · Model Training

Contracts for [Stage 4](../design/06-model-training.md). [Overview](../design/06-model-training.md) · **Specification** · [Implementation](../impl/training.md).

## Fold assignment (the `generate_folds` rule)

Generated **once** per `(seed_set, fold_seed)` — **and `target`** when stratified — over the **development** patients only, and written to a CSV. Every model in the sweep **reads the same CSV**; `train_run` takes the fold assignment as input and never re-derives it from a seed (so folds can't drift between models).

!!! warning "Stratified folds are target-dependent"
    Stratifying by the target makes the split depend on it, so "identical splits across models sharing a `seed_set`" holds only for runs sharing the **same target**. That is exactly what aligns folds across stains predicting the *same* score. Key the fold artifact by `(seed_set, target, fold_seed)`; use an unstratified (pure patient) split if target-independent folds are wanted.

| Field | Type | Notes |
|---|---|---|
| `patient_id` | str | development patients only; holdout excluded |
| `fold` | int | `0 … n_folds-1` |
| `split` | enum | `train` / `val` / `test` (per the active fold) |

Stored at `results/folds/…` and read by every run of that fold_seed; **not** in the bundle.

## Run record (`run.json`, one per run)

```text
run_id, run_family, model_experiment, bundle_id, cohort_id
target, subset, seed_set, fold_seed, model_seed
architecture { family, type, params }
hyperparameters { ... }
target_normalization { enabled, per fold: mean, std | class_map }   # if enabled, fit on the train fold only
metrics { train, val, test: {<metric>: value} }
checkpoints { fold_k: path }
membership_hash, git_commit, started_at, finished_at
```

`run_id` is the fully-realized per-model id (`f"{run_family}__fs{fold_seed}__ms{model_seed}"`);
`run_family` is the experiment-config run entry id (e.g. `"ki67_conch"`), shared by every
`fold_seed`/`model_seed` combination the seed sweep fans out into — what [evaluation](evaluation.md)
groups a sweep's contributing models by when writing one BEAM per biopsy.

!!! note "Target normalization is optional"
    Controlled by `target_normalization` (default **on**). When **on**, a regression model predicts in normalized space, so the per-fold `mean`/`std` (fit on that fold's train split) **must be stored** — evaluation de-normalizes back to label units, *per checkpoint* before a holdout ensemble averages. When **off** (e.g. to compare), the model trains and predicts directly in raw label units and nothing is stored or de-normalized.

!!! note "Optional per-patch axis features"
    Controlled by `append_axis_features` (default **off**). When **on**, each patch's biopsy-axis position — `axis_t` (normalized arc-length) and `axis_offset` (lateral distance), see [WSI Transformation spec](wsi-transformation.md#biopsy-axis-skeleton-curve) — is concatenated onto its embedding vector when the bundle's bags are loaded (`BagStore.from_bundle`), **not** baked into the cached embeddings file. `architecture.in_dim` is runtime-injected from the loaded embedding width, so this needs no other config change — just a bundle whose embeddings were produced after `axis_t`/`axis_offset` existed (re-run `preprocess embed` otherwise). The two raw scalars are appended as-is; any richer positional encoding (e.g. Fourier features) is a model-architecture extension, not a config option.

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
