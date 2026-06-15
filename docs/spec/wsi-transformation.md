1# Spec · WSI Transformation

Contracts for [Stage 2](../design/04-wsi-transformation.md). Overview · **Specification** · [Implementation](../impl/wsi-transformation.md).

## Coordinate frames

- **The reference stain defines the frame.** `reference_stain` (default `HE`, set in [`base.yaml`](../configs/base.md)) is what all others register to; for a patient, all *registered* variants (`rigid`, `elastic`) share its level-0 pixel frame.
- `raw` is each scan's own native frame.
- A registered variant ships a **transformation** mapping `raw → variant` (so raw-frame points — e.g. attention coordinates — can be placed on a registered underlay).

## Artifacts (per scan, per variant)

Outlines and the QC overlay are produced **within the registration step** (one rule). The tissue source is configurable — `tissue_method: valis` (default; the same masks VALIS registers on) or `hsv_otsu`.

!!! note "Method comparison is debug-only"
    The pipeline produces **one** authoritative outline per `(stain, variant)`, from the configured `tissue_method`, at a method-agnostic path (`…/outlines/{scan}__{variant}.geojson`). With `debug_compare_methods: true`, the step additionally runs the *other* method and writes its outline plus a side-by-side QC overlay to the **debug output folder** (`roots.debug`); these are never read by any downstream stage.

| Artifact | Form | Notes |
|---|---|---|
| Registered image | OME-TIFF | `rigid`, `elastic`; pyramidal, tiled, LZW |
| Rigid transform | `transform.json` | affine 3×3 (raw → rigid) + the reference-stain frame size |
| Elastic transform | displacement field ref + affine | non-rigid warp; stored as a VALIS/registration handle + the rigid prefix |
| Tissue outline | polygon array (+ GeoJSON) | per stain per variant; from the configured `tissue_method`, at a method-agnostic path; see [Outlines](../formats/outlines.md) |
| Cross-stain intersection | polygon array | per scan; from VALIS's `elastic` overlap mask |
| **QC overlay** | low-res **PNG** | per scan (outline on tissue) + one HE thumbnail with all outlines + the intersection |
| **Biopsy axis** | see below | per scan; the PCA longitudinal line |

## Biopsy axis (PCA line)

A per-scan longitudinal axis used for quartile subdivision and for ordering patches along the core. It is computed in the **`raw` frame** — the frame patches are drawn from for training — so a raw patch's quartile is well defined; registered-frame copies are derived for visualization only.

```text
biopsy_axis
  centroid        (2,) float    # x, y in the variant frame
  direction       (2,) float    # unit vector, 1st principal component
  t_min, t_max    float         # projection extent of tissue onto the axis
  length_px       float         # (t_max - t_min)
  length_mm       float         # length_px * mpp
  variance_ratio  float         # λ1 / (λ1 + λ2) — elongation sanity check
  quartile_cuts   (5,) float    # t at min, ¼, ½, ¾, max → Q1..Q4 segments
```

## Invariants

- Applying a scan's `rigid` transform to its `raw` outline reproduces the `rigid` outline within a small pixel tolerance.
- The cross-stain intersection is a subset of every stain's `elastic` outline for that scan.
- The biopsy axis `direction` is a unit vector; `quartile_cuts` are monotonic and split `[t_min, t_max]` into four equal-length segments.
- Outline vertices are in their variant's pixel frame; nothing is implied by filenames.
- Exactly one outline exists per `(stain, variant)` in the authoritative tree; its path does not encode the tissue method, and no downstream stage selects between methods (comparison outputs live only under `roots.debug`).

## Acceptance criteria

- Given a registered patient, every requested stain has an OME-TIFF, a transform, and an outline in each produced variant.
- `variance_ratio ≥ 0.9` for a well-formed biopsy core (flag below threshold — likely a fragmented or non-elongated sample).
- Re-running registration on identical inputs reproduces transforms within tolerance (deterministic seeds where the backend allows).
- Every patch later assigned a quartile has a projection `t ∈ [t_min, t_max]`.
