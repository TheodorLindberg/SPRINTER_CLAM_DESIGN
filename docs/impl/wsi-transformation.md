# Impl · WSI Transformation

Implementation notes for [Stage 2](../design/04-wsi-transformation.md). [Overview](../design/04-wsi-transformation.md) · [Specification](../spec/wsi-transformation.md) · **Implementation**.

Reference libraries: **VALIS** (registration **and** tissue masking), **OpenCV** (contours, rasterization), **scikit-image** (skeletonization), **scipy** (sparse shortest-path, connected components), **shapely** (outlines), **tifffile/zarr** (I/O), **numpy** (PCA).

> Design-level notes, not a prescriptive recipe. The contract is the [spec](../spec/wsi-transformation.md); library names below are suggestions, not commitments.

## Registration + outlines (one step)

Registration and outline extraction happen **together**: VALIS already segments tissue to drive registration and holds the transforms, so the same rule emits the registered images, the transforms, the outlines, the cross-stain intersection, and a QC overlay. The **`reference_stain`** (default `HE`, from [`base.yaml`](../configs/base.md)) is the reference; all stains register to it.

1. **Rigid** — feature-based alignment (e.g. VALIS with a DeDoDe / DISK+LightGlue detector), downscaled to a processing dimension. The per-slide affine `raw → rigid` is recovered by warping reference points and fitting a similarity transform, then saved to `transform.json`.
2. **Elastic (non-rigid)** — RANSAC-filtered matches produce a displacement field on top of the rigid prefix.
3. **Warp & save** — write a pyramidal OME-TIFF per variant. Keep the VALIS registrar so masks/points can be warped later.

!!! warning "Persist the transform for heatmaps"
    Training is on `raw` but heatmaps render on a registered underlay, so the `raw → variant` mapping must be **saved**, not just applied once. Persist the registration (affine + non-rigid field) so attention coordinates can be warped later. (Not needed when models train on already-registered images.)

## Tissue outlines

Two interchangeable tissue sources, behind one interface:

- **`valis`** (default) — VALIS's own tissue segmentation, the same masks it registers on, so the outline matches the registration.
- **`hsv_otsu`** — HSV-saturation + Otsu + morphology (closing → remove small objects → dilation → fill holes); a self-contained fallback that doesn't depend on VALIS internals.

Both share the same tail: get the tissue mask, warp it into the target frame (rigid / elastic / inverse for raw), trace contours, simplify, scale to level-0, and store as a polygon array (one outline per `(stain, variant)`; multiple components become a list). The **cross-stain intersection** comes from VALIS's overlap mask (or a shapely intersection of the `hsv_otsu` outlines).

Only the configured `tissue_method` is written into the main tree, at a method-agnostic path (`…/outlines/{scan}__{variant}.geojson`) that downstream stages read without ever branching on the method. When `debug_compare_methods` is on, the other method is additionally run and its outline written under the **debug folder** (`roots.debug`) alongside the comparison overlay; nothing downstream reads it.

## QC overlay (low-res PNG)

Per scan, draw the configured outline on a thumbnail (standard QC). Also emit one HE thumbnail with **all stains' outlines + the intersection** overlaid, to verify cross-stain alignment at a glance. When `debug_compare_methods` is on, a separate overlay with **both methods in different colours** is written to `roots.debug` for side-by-side judgement.

## Biopsy axis (skeleton curve)

The longitudinal **curve** of a core (possibly several islands), used to cut it into ordered regions and to give patches a continuous position. It is computed in the **`raw` frame** — the frame training patches are drawn from — so a raw patch's region is well defined; registered-frame copies are derived only for visualization ([spec](../spec/wsi-transformation.md#biopsy-axis-skeleton-curve)). The approach (`histomil.wsi_transformation.biopsy_axis`):

- Rasterize every kept tissue component onto one working-resolution mask. Components within `max_bridge_gap_mm` of each other are explicitly **bridged** — a line drawn between their closest boundary points — into one connected piece, deliberately not a single oversized morphological closing (whose structuring-element radius would have to match the gap and, at the gaps real fragmentation needs, closes right back into separate slivers — a discretization artifact, not a real bridge).
- **Skeletonize** the bridged mask (`skimage.morphology.skeletonize`); keep the largest connected skeleton component (any excluded fragment — too far to bridge — is logged, never failed on).
- Extract the skeleton's **longest path** (its graph diameter, via the standard two-pass shortest-path trick over its 8-connected pixel graph), smooth it, and resample to a compact ordered polyline — the curved axis `path`.
- The straight-line **principal eigenvector** of the tissue's covariance is *also* computed, purely as the `variance_ratio` diagnostic (largest eigenvalue / total — how line-like the core is); it no longer determines the stored geometry.
- A patch's **region** is found by projecting its centroid onto the *closest point* of `path` (not a dot product) and binning the resulting arc length by `quartile_cuts` (`n_segments + 1` equal arc-length cuts, `n_segments` configurable, default 4). The same projection's arc length (normalized to `[0, 1]`) and signed lateral offset are the two continuous per-patch scalars (`axis_t`, `axis_offset`) meant as positional information for the model.
- Store the centroid, direction (the endpoint-to-endpoint chord of `path` — a coarse summary), `path`, `t_min`/`t_max`, length (px and mm), variance ratio, `n_segments`, and `quartile_cuts`.
- When the skeleton can't be extracted (the `[wsi]` extra isn't installed, or the tissue is too small/degenerate — e.g. a near-circular blob with no clear long axis), **fall back to the straight PCA chord** between the extreme projections — never fails.

!!! note "Colour-PCA quartiles (alternative)"
    A separate option derives quartile *boundaries* from PCA on per-region stain-colour statistics rather than geometry. It emits the same outline/quartile format; geometry-PCA is the default.

## Outputs

From the single registration step: registered OME-TIFFs + transforms, per-variant per-stain outlines (polygon arrays + GeoJSON export), the cross-stain intersection, a low-res QC PNG per scan plus an HE overlay of all outlines, and the per-scan biopsy axis.
