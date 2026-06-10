# Cohorts — `cohorts.yaml`

The project-wide registry of **cohorts** — named, possibly multi-dataset groups of patients. Within a cohort, each patient has a **role**: `development` (where cross-validation happens) or `holdout` (locked away, evaluated once). Both [splits](seeds.md) and [bundles](preprocessing.md) derive from a cohort, so every stain/embedding bundle built from the same cohort shares aligned folds. See [Cohorts, roles, and splits](../design/02-data-model.md#cohorts-roles-and-splits).

The `holdout` block lists which patients take the holdout role; everyone else is `development`. Holdout can be an explicit patient list or a deterministic `fraction` + `seed`.

!!! note "Membership is frozen"
    Preprocessing resolves a cohort into a hashed membership manifest. If members change, the hash changes and dependent splits are flagged stale.

```yaml title="cohorts.yaml"
--8<-- "cohorts.yaml"
```
