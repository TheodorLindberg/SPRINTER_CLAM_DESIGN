# Stage 2 · `wsi_transformation.yaml`

Registration and tissue-outline detection: which [source variants](../design/02-data-model.md#source-variants) to produce, the registration backend, and outline options. See [Stage 2 · WSI Transformation](../design/04-wsi-transformation.md).

**Key fields**

- `variants` — which of `raw` / `rigid` / `elastic` to produce. Training uses `raw`; registered variants serve [heatmaps](../design/08-heatmaps.md).
- `reference_stain` — the stain all others register to (default `HE`, inherited from [`base.yaml`](base.md); override under `registration` only if needed).
- `outlines.tissue_method` — the single tissue method the pipeline produces and uses: `valis` (default) or `hsv_otsu`. The main outline path does **not** encode the method, so no downstream stage branches on it.
- `outlines.debug_compare_methods` — **debug only.** Also runs the *other* method and writes its outline plus a side-by-side overlay to the debug folder (`roots.debug`). The main pipeline still produces and consumes one outline per `(stain, variant)`; nothing downstream reads the comparison.
- `outlines.cross_stain_intersection` — the region with tissue in every stain, from the elastic overlap; saved per scan.
- `outlines.quartile_subdivision` — geometric split of the outline along the biopsy's long axis (see [Outlines](../formats/outlines.md)).
- `outlines.geojson_export` — also emit GeoJSON for TissUUmaps.

```yaml title="wsi_transformation.yaml"
--8<-- "wsi_transformation.yaml"
```
