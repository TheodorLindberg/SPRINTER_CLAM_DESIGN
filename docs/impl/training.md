# Impl · Model Training

Recipe for [Stage 4](../design/06-model-training.md). [Overview](../design/06-model-training.md) · [Specification](../spec/training.md) · **Implementation**.

Reference libraries: **scikit-learn** (stratified folds), **PyTorch** (training), **Optuna** (HPO), **pandas/pyarrow** (run records).

## Fold generation

```python
dev = [p for p in cohort.patients if p.role == "development"]
strata = bin(target_value(p) for p in dev)            # quantile bins for regression
skf = StratifiedKFold(n_splits=n_folds, shuffle=True, random_state=fold_seed)
folds = [test_idx for _, test_idx in skf.split(dev, strata)]
# per active fold i:  test = folds[i], val = folds[(i+1) % k], train = the rest
```

- Patient-level (all bags of a patient share a fold). Holdout patients are never in `dev`.
- Falls back to a shuffled split if a class has fewer than `n_folds` patients.

## Seed sweep

```python
for fold_seed in seed_set.fold_seeds:
    for model_seed in seed_set.model_seeds:
        run = train_run(bundle, target, architecture, hyperparameters,
                        fold_seed, model_seed)
        write_run_record(run)
aggregate_runs_parquet()
```

## Training a run

1. Build fold assignments (above). For each fold:
2. **Fit normalization on the train fold only** (regression target mean/std; class weights / bin balancing from the train split).
3. MIL forward: bag of embeddings → pooling (**CLAM** attention, **non-CLAM** mean-pool, or a **regression** head) → prediction.
4. Loss: Huber (regression) / cross-entropy (classification). Optimizer: AdamW. Early-stop on the val metric.
5. Log per-epoch train/val metrics; evaluate the fold's `test` split.
6. Save `fold_k` checkpoint + history.

Emit a [run record](../spec/training.md#run-record-runjson) with tags, metrics, checkpoint paths, `membership_hash`, and `git_commit`; append to `runs.parquet`.

### Label balancing

Computed per fold from the train split: class weights or weighted/over/under-sampling (classification); quantile-bin balancing (regression). Never fit on val/test.

### Architecture families

`family → type → params`. Reference adapter for **CLAM**: feed bundle bags (no filename/symlink hacks — read paths from the manifest) and the fold assignment; expose attention for [evaluation](../design/07-evaluation.md).

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
