# Outlines & geometry exports

Tissue outlines exist in two representations:

| Representation | Role |
|---|---|
| **Polygon arrays** (binary) | Source of truth used by the pipeline — in [BEAM](beam.md#outline) and bundles |
| **GeoJSON** (export) | Interactive viewing in TissUUmaps; never used for computation |

---

## Polygon array

An outline is a **list** of ordered vertex arrays in the WSI frame, one per disjoint tissue component, largest first. A biopsy fragmented during sectioning is several disjoint islands — every component above the minimum area is kept (`keep_islands: true`, default), not just the largest.

```text
outline
│
└─ components/ ············ one (Mᵢ, 2) float array per kept tissue component
   ├─ 0 ··················· largest component
   ├─ 1
   └─ …
```

The per-patch region (`quartile`, `1..n_segments`) and the two continuous per-patch scalars (`axis_t`, `axis_offset`) — the geometric split along the biopsy's curved long axis, bridged across islands — live on the **patch coordinates**, not on the outline itself; see [Embeddings & patches](embeddings-and-patches.md). It is **not** ground-truth localization — true per-region mapping is unavailable unless a pathologist-annotated region polygon is supplied (a documented future extension, reusing this same multi-polygon machinery).

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
