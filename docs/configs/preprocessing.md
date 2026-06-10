# Stage 3 · `preprocessing.yaml`

Two parts: heavy per-scan work (labels, patching, embedding — cached) and cheap **bundle assembly**. The `bundle` block names a [patient set](patient_sets.md) and a stain/embedding/variant; it materializes **one bundle containing every bag, each tagged with its cohort** (`development` / `holdout`) — not a separate bundle per cohort. **Augmentation also lives here** (it runs the foundation model on augmented patches). See [Stage 3 · Dataset Preprocessing](../design/05-dataset-preprocessing.md).

```yaml title="preprocessing.yaml"
--8<-- "preprocessing.yaml"
```
