# Stage 3 · `preprocessing.yaml`

Two parts: heavy per-scan work (labels, patching, embedding — cached) and cheap **bundle assembly**. A **bundle is a prepared cohort**: the `bundle` block names a [cohort](cohorts.md) and a stain/embedding/variant, and materializes every patient of that cohort, each bag tagged with its **role** (`development` / `holdout`). **Augmentation also lives here** (it runs the foundation model on augmented patches). See [Stage 3 · Dataset Preprocessing](../design/05-dataset-preprocessing.md).

```yaml title="preprocessing.yaml"
--8<-- "preprocessing.yaml"
```
