# Impl · WSI Transformation

Implementation notes for [Stage 2](../design/04-wsi-transformation.md). [Overview](../design/04-wsi-transformation.md) · [Specification](../spec/wsi-transformation.md) · **Implementation**.

Reference libraries: **VALIS** (registration **and** tissue masking), **OpenCV** (contours), **shapely** (outlines), **tifffile/zarr** (I/O), **numpy** (PCA).

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

When `emit_comparison` is on, both methods are written to **method-tagged paths** (`…/outlines/{scan}__{variant}__{method}.geojson`) so they coexist; downstream stages read only the configured `tissue_method`.

## QC overlay (low-res PNG)

Per scan, draw the outline(s) on a thumbnail. With comparison on, overlay **both methods in different colours** so they can be judged side by side. Also emit one HE thumbnail with **all stains' outlines + the intersection** overlaid, to verify cross-stain alignment at a glance.

## Biopsy axis (PCA line)

The longitudinal axis of a core, used to cut it into geometric quartiles. The approach:

- Take the tissue-pixel (or dense-outline) coordinates, scaled to level-0. The **principal eigenvector** of their covariance — the direction of largest variance — is the long axis. The **variance ratio** (largest eigenvalue / total) measures how line-like the core is.
- Project every point onto that direction to get a 1-D position along the axis. Its min and max give the core **length** (× `mpp` → mm), and four equal segments between them are the **quartile cuts**.
- A patch's quartile is found by projecting its centroid onto the axis and seeing which segment it falls in. Optionally, clipping the outline with lines perpendicular to the axis at each cut yields quartile **polygons** for visuals.
- Store the centroid, direction, `t_min`/`t_max`, length (px and mm), variance ratio, and quartile cuts.

!!! note "Curved cores — skeleton fallback"
    When the variance ratio is low (a bent or folded core), replace the straight axis with the mask **skeleton** as the path and use **arc-length** quartiles along it. Same stored fields, with the straight direction replaced by a polyline path.

!!! note "Colour-PCA quartiles (alternative)"
    A separate option derives quartile *boundaries* from PCA on per-region stain-colour statistics rather than geometry. It emits the same outline/quartile format; geometry-PCA is the default.

## Outputs

From the single registration step: registered OME-TIFFs + transforms, per-variant per-stain outlines (polygon arrays + GeoJSON export), the cross-stain intersection, a low-res QC PNG per scan plus an HE overlay of all outlines, and the per-scan biopsy axis.
