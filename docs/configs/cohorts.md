# Patient sets — `patient_sets.yaml`

The project-wide registry of **patient sets** — named, possibly multi-dataset collections of patients. Each set is partitioned into two **cohorts**: `development` (where cross-validation happens) and `holdout` (locked away, evaluated once). Both [splits](seeds.md) and [bundles](preprocessing.md) derive from a patient set, so every stain/embedding bundle built from the same set shares aligned folds. See [Cohorts vs. splits](../design/02-data-model.md#cohorts-vs-splits).

The `holdout` block defines the locked cohort; everything else is `development`. Hold-out can be an explicit patient list or a deterministic `fraction` + `seed`.

!!! note "Membership is frozen"
    Preprocessing resolves a patient set into a hashed membership manifest. If members change, the hash changes and dependent splits are flagged stale.

```yaml title="patient_sets.yaml"
--8<-- "patient_sets.yaml"
```
