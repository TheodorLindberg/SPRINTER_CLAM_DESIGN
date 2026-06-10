# Impl · WSI Transformation

Recipe for [Stage 2](../design/04-wsi-transformation.md). [Overview](../design/04-wsi-transformation.md) · [Specification](../spec/wsi-transformation.md) · **Implementation**.

Reference libraries: **VALIS** (registration **and** tissue masking), **OpenCV** (contours), **shapely** (outlines), **tifffile/zarr** (I/O), **numpy** (PCA).

## Registration + outlines (one step)

Registration and outline extraction happen **together**: VALIS already segments tissue to drive registration and holds the transforms, so the same rule emits the registered images, the transforms, the outlines, the cross-stain intersection, and a QC overlay. The **`reference_stain`** (default `HE`, from [`base.yaml`](../configs/base.md)) is the reference; all stains register to it.

1. **Rigid** — `Valis(..., reference_img_f=<reference_stain scan>, feature_detector_cls=DeDoDeFD, micro_rigid_registrar_cls=MicroRigidRegistrar)` (or DISK + LightGlue for feature-rich samples), downscaled to `max_processed_image_dim_px`. Per slide, the affine `raw → rigid` is recovered by warping reference points (`slide.warp_xy(pts, non_rigid=False)` → `SimilarityTransform.estimate`) and saved to `transform.json`.
2. **Elastic (non-rigid)** — RANSAC-filtered matches → displacement field on top of the rigid prefix.
3. **Warp & save** — `warp_and_save_slides(dir, compression="lzw")` → pyramidal OME-TIFF per variant. Keep the VALIS registrar so masks/points can be warped later.

!!! warning "Persist the transform for heatmaps"
    Training is on `raw` but heatmaps render on a registered underlay, so the `raw → variant` mapping must be **saved**, not just applied once. Persist the VALIS registration (affine + non-rigid field) so attention coordinates can be warped later with `warp_tools.warp_xy`. (Not needed when models train on already-registered images.)

## Tissue outlines

Two interchangeable tissue sources, behind one interface:

- **`valis`** (default) — VALIS's own tissue segmentation (`valis.preprocessing`), the same masks it registers on, so the outline matches the registration.
- **`hsv_otsu`** — HSV-saturation + Otsu + morphology (`closing` → `remove_small_objects` → `dilation` → `fill_holes`); a self-contained fallback that doesn't depend on VALIS internals.

Both share the mask → outline tail:

1. Get the tissue mask (either method). Warp it into the target frame with the registrar (`non_rigid=False` → `rigid`; `non_rigid=True` → `elastic`; inverse → `raw`).
2. `cv2.findContours` → simplify → scale to level-0 → polygon array (multiple components → list). One outline per `(stain, variant)`.
3. **Cross-stain intersection** — VALIS's **overlap mask** (or a shapely intersection of the `hsv_otsu` outlines) → contour → polygon.

### Comparing methods (development)

Write each method to a **method-tagged path** so they coexist; downstream reads only `tissue_method`:

```python
methods = {cfg.tissue_method} | ({other(cfg.tissue_method)} if cfg.emit_comparison else set())
for m in methods:
    outline = mask_to_outline(tissue_mask(m, slide), variant)
    save_geojson(outline, f"…/outlines/{scan}__{variant}__{m}.geojson")
```

## QC overlay (low-res PNG)

Per scan, draw the outline(s) on a thumbnail. When comparison is on, overlay **both methods in different colours** so they can be judged side by side:

```python
thumb = registrar.get_thumbnail(slide, variant)              # low-res RGB
colors = {"valis": (255,0,0), "hsv_otsu": (0,180,0)}
for m, outline in outlines.items():
    cv2.polylines(thumb, [to_lowres(outline).astype(int)], True, colors[m], 2)
cv2.imwrite(f"{scan}__{variant}_outline.png", thumb)
```

Also emit one HE thumbnail with **all stains' outlines + the intersection** overlaid, to verify cross-stain alignment at a glance.

## Biopsy axis (PCA line)

The longitudinal axis from the registered tissue mask. Straight-line PCA suits mostly-straight cores; see the skeleton fallback for curved ones.

```python
# pts: (M,2) tissue-pixel (or dense outline) coords at low res, scaled to level-0
mu   = pts.mean(0)
cov  = np.cov((pts - mu).T)            # 2x2
val, vec = np.linalg.eigh(cov)         # ascending
direction = vec[:, -1]                 # largest eigenvalue → long axis
variance_ratio = val[-1] / val.sum()

t = (pts - mu) @ direction             # 1-D position along the axis
t_min, t_max = t.min(), t.max()
length_px = t_max - t_min
length_mm = length_px * mpp
quartile_cuts = t_min + np.linspace(0, 1, 5) * length_px
```

- **Quartile assignment**: project a patch centroid onto `direction`; its `t` falls in one of the four `quartile_cuts` segments → `Q1..Q4`.
- **Quartile polygons** (optional, for visuals): clip the outline with lines perpendicular to `direction` at each interior cut.
- Store `{centroid: mu, direction, t_min, t_max, length_px, length_mm, variance_ratio, quartile_cuts}`.

!!! note "Curved cores — skeleton fallback"
    When `variance_ratio` is low (bent/folded core), replace the straight axis with the mask **skeleton** (`skimage.morphology.skeletonize`) as the path and use **arc-length** quartiles along it. Same stored fields, with `direction` replaced by a polyline path.

!!! note "Colour-PCA quartiles (alternative)"
    A separate option derives quartile *boundaries* from PCA on per-region stain colour statistics rather than geometry. Keep it as an alternate quartile source emitting the same outline/quartile format; geometry-PCA is the default.

## Outputs

From the single registration step: registered OME-TIFFs + transforms, per-variant per-stain outlines (polygon arrays + GeoJSON export), the cross-stain intersection, **a low-res QC PNG per scan + an HE overlay of all outlines**, and the per-scan biopsy axis.
