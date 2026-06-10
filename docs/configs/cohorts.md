# Cohorts — `cohorts.yaml`

The project-wide registry of **cohorts** — named, possibly multi-dataset groups of patients. Within a cohort each patient has a **role**: `development` (where cross-validation happens) or `holdout` (locked away, scored once). Both [splits](seeds.md) and [bundles](preprocessing.md) derive from a cohort, so every bundle built from it shares the same folds. See [Cohorts, roles, and splits](../design/02-data-model.md#cohorts-roles-and-splits).

**Key fields**

- `members` — patients in the cohort, listed per dataset source (`patients: all` or an explicit list); pooling across sources is what `(dataset_id, patient_id)` enables.
- `holdout` — which patients take the holdout role, as an `explicit` list **or** a deterministic `fraction` + `seed`. Everyone else is `development`.

!!! note "Membership is frozen"
    Preprocessing resolves a cohort into a hashed membership manifest. If members change, the hash changes and dependent splits are flagged stale.

```yaml title="cohorts.yaml"
--8<-- "cohorts.yaml"
```
