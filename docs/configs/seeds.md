# Split registry — `seeds.yaml`

The project-wide registry of named seed/split configurations, indexed by name under `seed_sets`. Each set references a [cohort](cohorts.md) (not a dataset) — folds are computed over that cohort's **development** patients, so holdout patients never enter a fold. A run picks one with `seed_set:` in [`model_experiment.yaml`](model_experiment.md); every model using the same name gets identical splits. If the cohort's frozen membership changes, the pipeline warns that the splits are stale.

```yaml title="seeds.yaml"
--8<-- "seeds.yaml"
```
