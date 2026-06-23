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
| **Biopsy axis** | see below | per scan; the skeleton/medial-axis curve |

Tissue outlines keep **every** disjoint component above the minimum area (`keep_islands: true`, default) — a biopsy fragmented during sectioning is several disjoint polygons, not just the largest — rather than discarding all but the largest. An outline is therefore a **list** of polygons, largest first (see [Outlines](../formats/outlines.md)).

## Biopsy axis (skeleton curve)

A per-scan longitudinal **curve** (not a straight line) used for region subdivision and for ordering/positioning patches along the core. It is computed in the **`raw` frame** — the frame patches are drawn from for training — so a raw patch's region is well defined; registered-frame copies are derived for visualization only. It is fit across **every kept tissue component**: islands within `max_bridge_gap_mm` of each other are bridged into one continuous axis; an island farther away is excluded (logged, never failed on).

```text
biopsy_axis
  centroid        (2,) float    # x, y — coarse chord summary, not the canonical geometry
  direction       (2,) float    # unit vector, endpoint-to-endpoint chord of `path`
  path            (P, 2) float  # the ordered curve itself, P >= 2 (2 = straight-chord fallback)
  t_min, t_max    float         # arc-length extent along `path` (t_min is always 0)
  length_px       float         # (t_max - t_min), arc length along the curve
  length_mm       float         # length_px * mpp
  variance_ratio  float         # straight-line PCA λ1 / (λ1 + λ2) — elongation diagnostic only
  n_segments      int           # number of ordered regions (default 4; configurable)
  quartile_cuts   (n_segments + 1,) float   # arc-length cuts → region 1..n_segments
```

A patch's **region** (`quartile`) is found by projecting its centroid onto the *closest point* of `path` (not a straight-line dot product) and binning the resulting arc length by `quartile_cuts`. That same projection also yields two continuous per-patch scalars carried alongside the region — `axis_t` (arc-length position, normalized to `[0, 1]`) and `axis_offset` (signed lateral distance from the curve) — meant as positional information for the model (e.g. Fourier/positionally encoded), not just the coarse region bucket. The field is still named `quartile` for historical/UX reasons even though `n_segments` need not be 4.

## Invariants

- Applying a scan's `rigid` transform to its `raw` outline reproduces the `rigid` outline within a small pixel tolerance.
- The cross-stain intersection is a subset of every stain's `elastic` outline for that scan.
- The biopsy axis `path` has at least 2 vertices; `quartile_cuts` has exactly `n_segments + 1` monotonically non-decreasing values spanning `[t_min, t_max]`.
- Closest-point-on-`path` projection is, by construction, always within `[t_min, t_max]` — a patch's region is `0` ("unassigned") only when no axis was computed at all, never from the projection itself.
- Outline vertices are in their variant's pixel frame; nothing is implied by filenames.
- Exactly one outline exists per `(stain, variant)` in the authoritative tree; its path does not encode the tissue method, and no downstream stage selects between methods (comparison outputs live only under `roots.debug`).

## Acceptance criteria

- Given a registered patient, every requested stain has an OME-TIFF, a transform, and an outline (every kept tissue component) in each produced variant.
- `variance_ratio ≥ 0.9` for a well-formed, non-fragmented biopsy core (flag below threshold — likely a fragmented or non-elongated sample; the axis is still computed, not refused).
- Re-running registration on identical inputs reproduces transforms within tolerance (deterministic seeds where the backend allows).
- Every patch later assigned a region has an arc-length projection `t ∈ [t_min, t_max]`.
