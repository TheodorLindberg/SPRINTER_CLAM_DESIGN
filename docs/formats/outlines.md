# Outlines & geometry exports

Tissue outlines exist in two representations:

- **Polygon arrays (binary)** — the source of truth used by the pipeline (in [BEAM](beam.md#outline) and bundles).
- **GeoJSON (export)** — produced for interactive viewing in TissUUmaps. The pipeline never computes from the GeoJSON.

## Polygon array

An outline is an array of vertices in the WSI frame:

```text
polygon : (M, 2) float    # ordered x, y vertices
```

When several disjoint tissue regions exist, an outline is a list of such arrays.

## Quartile subdivision

An outline may be divided into four sub-polygons along the biopsy's long axis:

```text
quartiles : { q0, q1, q2, q3 }   # each an (Mi, 2) float polygon
```

This is a **geometric** split, provided so the per-quartile Ki-67 / PSA scores can be associated with approximate regions of the biopsy. It is not ground-truth region localization — true per-region mapping is unavailable.

## Variants

Outlines are produced per source variant in [WSI Transformation](../design/04-wsi-transformation.md):

- Raw outline — per stain
- Rigid-registered outline — per stain
- Elastic-registered outline — per stain
- Cross-stain intersection outline — per scan (from the overlapping elastic registration)

## GeoJSON export

For TissUUmaps, polygons are exported as GeoJSON `Polygon` / `MultiPolygon` features. Patch geometry can be exported the same way, carrying attention values as feature properties (used by [heatmaps](../design/08-heatmaps.md)).
