# Outlines & geometry exports

Tissue outlines exist in two representations:

| Representation | Role |
|---|---|
| **Polygon arrays** (binary) | Source of truth used by the pipeline — in [BEAM](beam.md#outline) and bundles |
| **GeoJSON** (export) | Interactive viewing in TissUUmaps; never used for computation |

---

## Polygon array

An outline is an ordered array of vertices in the WSI frame. When several disjoint tissue regions exist, an outline is a list of such arrays.

```text
outline
│
├─ polygon ················ (M, 2) float   ordered x, y vertices · WSI frame
│
└─ quartiles/ ············· optional · geometric split along biopsy long axis
   ├─ q0 ················· (M₀, 2) float
   ├─ q1 ················· (M₁, 2) float
   ├─ q2 ················· (M₂, 2) float
   └─ q3 ················· (M₃, 2) float
```

!!! note "Quartiles are an approximation"
    The quartile split is **geometric** (along the biopsy's long axis), provided so the per-quartile Ki-67 / PSA scores can be associated with approximate regions. It is **not** ground-truth localization — true per-region mapping is unavailable.

---

## Variants

Outlines are produced per source variant in [WSI Transformation](../design/04-wsi-transformation.md):

| Outline | Granularity |
|---|---|
| Raw outline | per stain |
| Rigid-registered outline | per stain |
| Elastic-registered outline | per stain |
| Cross-stain intersection | per scan (from the overlapping elastic registration) |

---

## GeoJSON export

For TissUUmaps, polygons are exported as GeoJSON `Polygon` / `MultiPolygon` features. Patch geometry is exported the same way, carrying attention values as feature properties (used by [heatmaps](../design/08-heatmaps.md)).
