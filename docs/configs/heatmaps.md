# Stage 6 · `heatmaps.yaml`

Renders attention/prediction heatmaps from BEAM files over a chosen underlay. See [Stage 6 · Heatmaps](../design/08-heatmaps.md).

**Key fields**

- `source_variant` — underlay alignment: `raw`, `rigid`, or `elastic`.
- `attention_type` — `rank` (robust default), `sigmoid`, or `raw`.
- `colormap` — perceptually uniform (e.g. viridis).
- `render` — tissue masking, outline, scale bar (from mpp), and provenance caption.
- `outputs` — static PNG and/or a TissUUmaps GeoJSON layer carrying attention as properties.

!!! warning "Coordinates are transformed to the underlay"
    Attention lives in the **raw** frame, so coordinates are pushed through the chosen variant's transformation matrix before overlay.

```yaml title="heatmaps.yaml"
--8<-- "heatmaps.yaml"
```
