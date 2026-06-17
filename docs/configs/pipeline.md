# Pipeline config — `pipeline.yaml`

Data-prep stage defaults in one file, as **independent sections**: WSI transformation, preprocessing, and reports. Loaded with [`base.yaml`](base.md); a single stage target reads only its own section, so the three stay editable independently. See [Stage 2](../design/04-wsi-transformation.md), [Stage 3](../design/05-dataset-preprocessing.md), and [Reports](../design/11-reports.md).

**Sections**

- `wsi_transformation` — variants to produce, registration backend, outline + quartile options (`reference_stain` comes from `base.yaml`).
- `preprocessing` — `labels.sources` / `labels.derived`, `patching` (size, resolution, overlap), `embedding.model`, and the `bundle` (a [prepared cohort](../design/05-dataset-preprocessing.md#bundle-preparation) for one stain/embedding/variant — run repeatedly for different stains/embeddings).
- `reports` — plot backend, standalone CSS, HPO segregation, table export, section toggles.

```yaml title="pipeline.yaml"
--8<-- "pipeline.yaml"
```
