# Stage 4 · Model Training

Trains MIL models from a bundle. A **model experiment** defines the runs, which fan out over the seed sweep; HPO is a separate search. (Evaluation-only runs skip this stage and go straight to [Stage 5](07-evaluation.md).)

> **In** a bundle (`development` subset) + a `seed_set` · **Out** per-fold checkpoints, metrics, run records

```mermaid
flowchart TD
    EXP[Model experiment\nshared defaults + runs] --> FG[Fold generation\nseed_set → seeds.yaml]
    FG --> HPO[HPO trials\nsegregated · top-N kept]
    FG --> SW[Seed sweep\nfold seed × model seed]
    HPO -->|promote best N| SW
    SW --> TF[Training function]
```

---

## Model experiments and runs

A [model experiment config](../configs/model_experiment.md) holds shared `defaults` plus an explicit list of **runs**, each a named variation (a different bundle, or different hyperparameters) with a `run_id`. Each run still **fans out over the seed sweep**. HPO is a separate config ([`hpo.yaml`](../configs/hpo.md)). Every run emits a **run record** aggregated into `runs.parquet` (see [Reports](11-reports.md)).

A run operates on its bundle at the **`development`** subset (or `all` for a final retrain after holdout). Folds are assigned only over `development` patients; `holdout` patients are filtered out and never enter a fold.

## Fold generation

Generates folds from fold seeds over the **development patients** of the bundle's [cohort](../configs/cohorts.md). A project-wide **split registry** ([`seeds.yaml`](../configs/seeds.md)) holds every seed/split configuration under `seed_sets`, indexed by name; a run picks one with `seed_set:`, so all models sharing that name get identical splits — and because folds are assigned to *patients*, every stain/embedding bundle from the same cohort inherits the same split. (The bundle's cohort and the `seed_set`'s cohort must match — validated.)

!!! warning "Membership changes invalidate splits"
    Splits are computed against the cohort's frozen, hashed membership. If membership changes (the hash differs), the pipeline raises a prominent warning that splits are stale.

---

## Seed sweep

Runs the training function across all combinations of **fold seed** and **model seed**, varied independently:

- The **model seed** controls random weight initialization.
- The **fold seed** controls the split.

It aggregates results and emits train/test/loss plots plus a CSV (or similar) of performance and metrics, for a clear view of model behavior.

---

## Hyperparameter optimization

Supports **grid search** and a **Bayesian optimizer** (e.g. Optuna / TPE), declared as a search space in its **own [`hpo.yaml`](../configs/hpo.md)** — separate from the model experiment, hundreds of trials, zero extra config files.

HPO is **kept apart from the seed sweep**, because HPO models are rarely revisited while the sweep models are the ones you keep:

- HPO outputs live under `results/experiments/{name}/hpo/` with their **own index**; the seed-sweep models live under `sweep/`, easy to find.
- `reports.yaml → hpo.keep_checkpoints` decides storage (`all` / `top_n` / `none`); by default only the **top-N** are retained.
- **Workflow:** HPO explores → promote the best N hyperparameters → run a **seed sweep** on them. The sweep is the durable result; HPO is exploratory.

---

## Training function

### Input

- Path to bundle (exact schema TBD).
- Folds CSV.
- Model seed.
- Hyperparameters (learning rate, regularization, dropout rates).
- Model architecture — **family → type → specific parameters** (hidden layer sizes, layer count, …). Predefined families include regression and CLAM / non-CLAM, covering attention mechanisms vs. mean pooling.
- Training name.
- Output directory.

### Output

- Model checkpoint per fold.
- Training history (train + validation), with metrics by task type.
- Test performance.
- Folds used.
- Training logs.

---

## Label balancing

Training supports correcting for imbalanced targets, configured per run:

- **Classification** — class weights, or weighted/over/under-sampling of bags.
- **Regression** — bin the target and balance across bins (so rare high/low scores are not drowned out).

Balancing is applied **per fold, from the training split only** — consistent with the [no-fitted-statistics rule](05-dataset-preprocessing.md) — so it never leaks distribution information from validation or held-out data.

---

## Augmentation

Augmentation is **not** a training concern in this pipeline. Meaningful histology augmentation (flips, rotations, stain/color jitter) changes the pixels the embedding model sees, so it must run the **foundation model** on the augmented patches — that happens in [Dataset Preprocessing](05-dataset-preprocessing.md#augmentation), where each augmented variant is embedded and cached as its own set.

Training only decides **whether to sample** those augmented embedding sets (`use_augmented_embeddings` in [`model_experiment.yaml`](../configs/model_experiment.md)).

---

## Metrics by label type

The metric set is selected **per label according to its type**, so a single sweep can report regression and classification targets side by side.

| Label type | Metrics |
|---|---|
| Regression | MAE, Spearman, R², Huber loss |
| Binary / classification | AUROC, accuracy, F1 (and related), as appropriate |
