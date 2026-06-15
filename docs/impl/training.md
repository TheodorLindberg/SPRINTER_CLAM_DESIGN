# Impl · Model Training

Implementation notes for [Stage 4](../design/06-model-training.md). [Overview](../design/06-model-training.md) · [Specification](../spec/training.md) · **Implementation**.

Reference libraries: **scikit-learn** (stratified folds), **PyTorch** (training), **Optuna** (HPO), **pandas/pyarrow** (run records).

> Design-level notes, not a prescriptive recipe. The contract is the [training spec](../spec/training.md).

## Fold generation (the `generate_folds` rule)

Folds are generated **once**, by their own rule, and written to a CSV. Training **reads** that CSV — it never re-derives a split from a seed, so every model in the sweep trains on byte-identical folds.

- Split the **development** patients with a stratified k-fold (quantile bins for regression targets), seeded by `fold_seed`. For fold *i*: `test` = fold *i*, `val` = the next fold, `train` = the rest.
- Patient-level — all bags of a patient share a fold. Holdout patients are never in the development set.
- Fall back to a shuffled split if a class has fewer than `n_folds` patients.

## Seed sweep

The fold CSV is **passed into** `train_run`, not a `fold_seed` for it to regenerate. For each `fold_seed`, read its pre-generated fold CSV; for each `model_seed`, train a run with those folds and write a [run record](../spec/training.md#run-record-runjson-one-per-run). All records aggregate into `runs.parquet`.

## Bag I/O — load once, reuse across the sweep

A bundle's embeddings are **static** after preprocessing, so they should be read from disk once and reused — not re-read per epoch, per fold, or per run. The seed sweep trains `fold_seeds × model_seeds` models over the same bundle and [evaluation](evaluation.md) reads it yet again; re-reading each time repeats identical I/O dozens of times (the training analog of the per-patient inference trap).

- Load the bundle's bags into memory **once** and share that store across every run in the sweep and across evaluation. Bags are feature vectors, not pixels (~20–60 MB/scan), so a bundle often fits in RAM; otherwise memory-map the H5.
- Per-fold balancing and sampling select **indices** into this static store — they never copy or re-read embeddings.
- Optionally pre-pack all bags into one contiguous array + a per-bag offset index, so a run does a handful of large reads instead of thousands of small H5 opens; pin memory for fast host→device transfer.

## Training a run

A run takes the fold assignment as input. For each fold:

1. If `target_normalization` (default on), **fit the target mean/std on the train fold only** and store it; if off, train in raw label units. Class weights / bin balancing are always from the train split.
2. MIL forward: bag of embeddings → pooling → prediction.
3. Loss + optimizer per family (see below); early-stop on the val metric.
4. Log per-epoch train/val metrics; evaluate the fold's `test` split.
5. Save the fold checkpoint + history.

Emit a [run record](../spec/training.md#run-record-runjson-one-per-run) with tags, metrics, checkpoint paths, `membership_hash`, and `git_commit`; append to `runs.parquet`.

### Augmented bags

If `use_augmented_embeddings`, the bundle's `augmented`-tagged bags are added to the **train split only**, in their patient's fold. They never enter `val` / `test` / `holdout` — otherwise metrics would be optimistic and the split would leak.

### Label balancing

Computed per fold from the train split: class weights or weighted/over/under-sampling (classification); quantile-bin balancing (regression). Never fit on val/test.

### Architecture families

`family → type → params`:

- **`clam`** — **classification** MIL (attention). Real knobs: `model_type` (`clam_sb` / `clam_mb`), `model_size`, `bag_loss`, `inst_loss`, `bag_weight`, instance-cluster `B`, dropout. The adapter feeds bundle bags **read from the manifest** (no filename/symlink hacks) plus the fold assignment, and exposes attention for [evaluation](../design/07-evaluation.md).
- **`non_clam`** — attention-free mean-pool MIL baseline.
- **`regression`** — a continuous-target MIL head (Huber/MSE). **New** — CLAM is classification-only, so a regression target either uses this head or is binned into classes for a CLAM run.

!!! note "Proven vs. new"
    The CLAM classification path and the embedding/bag inputs are proven in the reference code. The **regression head**, alternative optimizers, and **Optuna HPO** below are new in this design — feasible, but not yet exercised end-to-end.

## HPO (separate)

An Optuna study (TPE sampler) whose objective samples a point from `hpo.space` and returns the mean validation metric across a CV subset of fold seeds, optimised over `n_trials`. The best `promote_top_n` configurations are promoted into a seed sweep. Outputs go under `results/experiments/{name}/hpo/` with their own index; checkpoint retention follows `reports.yaml → hpo.keep_checkpoints`.
