# Stage 2 · `wsi_transformation.yaml`

Registration and tissue-outline detection: which [source variants](../design/02-data-model.md#source-variants) to produce, the registration backend, and outline options. See [Stage 2 · WSI Transformation](../design/04-wsi-transformation.md).

**Key fields**

- `variants` — which of `raw` / `rigid` / `elastic` to produce. Training uses `raw`; registered variants serve [heatmaps](../design/08-heatmaps.md).
- `registration.reference_stain` — the stain IHC stains are aligned to (H&E).
- `outlines.tissue_method` — tissue source used downstream: `valis` (default) or `hsv_otsu`.
- `outlines.emit_comparison` — during development, also produce the *other* method's outline (tagged in the path) and overlay both in the QC PNG, so the two can be compared. Only the `tissue_method` outline feeds later stages.
- `outlines.cross_stain_intersection` — the region with tissue in every stain, from the elastic overlap; saved per scan.
- `outlines.quartile_subdivision` — geometric split of the outline along the biopsy's long axis (see [Outlines](../formats/outlines.md)).
- `outlines.geojson_export` — also emit GeoJSON for TissUUmaps.

```yaml title="wsi_transformation.yaml"
--8<-- "wsi_transformation.yaml"
```
