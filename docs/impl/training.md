# Impl · Model Training

Recipe for [Stage 4](../design/06-model-training.md). [Overview](../design/06-model-training.md) · [Specification](../spec/training.md) · **Implementation**.

Reference libraries: **scikit-learn** (stratified folds), **PyTorch** (training), **Optuna** (HPO), **pandas/pyarrow** (run records).

## Fold generation (the `generate_folds` rule)

Folds are generated **once**, by their own rule, and written to a CSV. Training **reads** that CSV — it never re-derives a split from a seed, so every model in the sweep trains on byte-identical folds.

```python
# generate_folds — runs once per (seed_set, target, fold_seed); writes folds.csv
dev = [p for p in cohort.patients if p.role == "development"]
strata = bin(target_value(p) for p in dev)            # quantile bins for regression
skf = StratifiedKFold(n_splits=n_folds, shuffle=True, random_state=fold_seed)
folds = [test_idx for _, test_idx in skf.split(dev, strata)]
# per fold i: test = folds[i], val = folds[(i+1) % k], train = the rest
write_csv(f"folds/{seed_set}/{target}/{fold_seed}.csv", folds)   # patient → fold/split
```

- Patient-level (all bags of a patient share a fold). Holdout patients are never in `dev`.
- Falls back to a shuffled split if a class has fewer than `n_folds` patients.

## Seed sweep

The fold CSV is **passed into** `train_run` — not a `fold_seed` for it to regenerate.

```python
for fold_seed in seed_set.fold_seeds:
    folds = read_csv(f"folds/{seed_set}/{target}/{fold_seed}.csv")   # pre-generated
    for model_seed in seed_set.model_seeds:
        run = train_run(bundle, target, architecture, hyperparameters,
                        folds=folds, model_seed=model_seed)          # folds in, not regenerated
        write_run_record(run)
aggregate_runs_parquet()
```

## Training a run

`train_run(..., folds, model_seed)` takes the fold assignment as input. For each fold in it:
2. If `target_normalization` (default on), **fit the target mean/std on the train fold only** and store it; if off, train in raw label units. Class weights / bin balancing are always from the train split.
3. MIL forward: bag of embeddings → pooling → prediction.
4. Loss + optimizer per family (see below). Early-stop on the val metric.
5. Log per-epoch train/val metrics; evaluate the fold's `test` split.
6. Save `fold_k` checkpoint + history.

Emit a [run record](../spec/training.md#run-record-runjson) with tags, metrics, checkpoint paths, `membership_hash`, and `git_commit`; append to `runs.parquet`.

### Augmented bags

If `use_augmented_embeddings`, the bundle's `augmented`-tagged bags are added to the **train split only**, in their patient's fold. They never enter `val` / `test` / `holdout` — otherwise metrics would be optimistic and the split would leak.

### Label balancing

Computed per fold from the train split: class weights or weighted/over/under-sampling (classification); quantile-bin balancing (regression). Never fit on val/test.

### Architecture families

`family → type → params`:

- **`clam`** — **classification** MIL (attention). Real knobs: `model_type` (`clam_sb`/`clam_mb`), `model_size`, `bag_loss: ce`, `inst_loss: svm`, `bag_weight`, instance-cluster `B`, `drop_out`; optimizer `adam`. The adapter feeds bundle bags **read from the manifest** (no filename/symlink hacks) plus the fold assignment, and exposes attention for [evaluation](../design/07-evaluation.md).
- **`non_clam`** — attention-free mean-pool MIL baseline.
- **`regression`** — a continuous-target MIL head (Huber/MSE, AdamW). **New** — CLAM is classification-only, so a regression target either uses this head or is binned into classes for a CLAM run.

!!! note "Proven vs. new"
    The CLAM classification path and the embedding/bag inputs are proven in the reference code. The **regression head**, **AdamW**, and **Optuna HPO** below are new additions in this design — feasible, but not yet exercised end-to-end.

## HPO (separate)

```python
study = optuna.create_study(direction="maximize", sampler=TPESampler(seed=...))
def objective(trial):
    hp = sample(trial, hpo.space)
    return mean(val_metric(train_run(bundle, target, arch, hp, fold_seed, model_seed))
                for fold_seed in seed_set.fold_seeds[:hpo.cv_subset])
study.optimize(objective, n_trials=hpo.n_trials)
promote_top_n(study, hpo.top_n)          # → seed-sweep these in a model experiment
```

Outputs go under `results/experiments/{name}/hpo/` with their own index; keep `top_n` checkpoints per `reports.yaml`.
