# Stage 3 · `preprocessing.yaml`

Two parts: **heavy per-scan work** (labels, patching, embedding — cached) and cheap **bundle assembly**. See [Stage 3 · Dataset Preprocessing](../design/05-dataset-preprocessing.md).

**Key fields**

- `labels.derived` — labels computed from the raw scores (averages, thresholds, …); extendable per dataset. See the [label model](../design/02-data-model.md#label-model).
- `patching` — patch size, resolution, and overlap (stride derives from overlap).
- `embedding.cache` — the [content-addressed cache](../formats/embeddings-and-patches.md#content-addressed-cache-key); only cache misses are embedded.
- `embedding.augmentation` — image-space augmentation that **runs the foundation model** on augmented patches and caches each variant separately ([why it lives here](../design/05-dataset-preprocessing.md#augmentation)).
- `bundle` — **a bundle is a [prepared cohort](../design/05-dataset-preprocessing.md#bundle-preparation)** for one stain/embedding/variant; every patient is included, tagged by `role`. Run it repeatedly for different stains/embeddings.

```yaml title="preprocessing.yaml"
--8<-- "preprocessing.yaml"
```
