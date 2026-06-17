# Split registry — `seeds.yaml`

The project-wide registry of named seed/split configurations, indexed under `seed_sets`. Each set references a [cohort](cohorts.md) — folds are computed over that cohort's **development** patients, so holdout patients never enter a fold. A run picks one with `seed_set:` in the [experiment config](experiment.md); every run using the same name gets identical splits.

**Key fields**

- `cohort` — the cohort whose development patients are split.
- `split_level` — the unit splits respect (`patient` prevents same-patient leakage across folds).
- `n_folds` — number of cross-validation folds.
- `fold_seeds` — each seed produces a distinct fold partition; the [seed sweep](../design/06-model-training.md#seed-sweep) varies this.
- `model_seeds` — each seed produces a distinct weight initialization, varied independently of the fold seed.

!!! note "Stale-split detection"
    Splits are computed against the cohort's frozen membership hash. If membership changes, the pipeline warns that splits are stale.

```yaml title="seeds.yaml"
--8<-- "seeds.yaml"
```
