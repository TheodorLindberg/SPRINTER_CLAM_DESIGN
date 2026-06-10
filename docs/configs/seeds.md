# Split registry — `seeds.yaml`

The project-wide registry of named seed/split configurations, indexed by name under `seed_sets`. Each set references a [patient set](patient_sets.md) (not a dataset) — folds are computed over that set's **development** cohort, so holdout patients never enter a fold. A model picks one with `seed_set:` in [`training.yaml`](training.md); every model using the same name gets identical splits. If the patient set's frozen membership changes, the pipeline warns that the splits are stale.

```yaml title="seeds.yaml"
--8<-- "seeds.yaml"
```
