# Impl · WSI Transformation

Recipe for [Stage 2](../design/04-wsi-transformation.md). [Overview](../design/04-wsi-transformation.md) · [Specification](../spec/wsi-transformation.md) · **Implementation**.

Reference libraries: **VALIS** (registration), **OpenCV** + **scikit-image** (masks), **shapely** (outlines), **tifffile/zarr** (I/O), **numpy** (PCA).

## Registration (VALIS)

HE is the reference slide; all stains register to it.

1. **Rigid** — `Valis(..., reference_img_f=HE, feature_detector_cls=DeDoDeFD, micro_rigid_registrar_cls=MicroRigidRegistrar)`, downscaling feature detection to `max_processed_image_dim_px ≈ 500`. Produces the affine `raw → rigid`.
2. **Elastic (non-rigid)** — DISK keypoints + LightGlue matcher (SuperPoint + SuperGlue as a fallback for feature-rich samples), RANSAC-filtered. Produces the displacement field on top of the rigid prefix.
3. **Warp & save** — `warp_and_save_slides(dir, compression="lzw")` → pyramidal OME-TIFF per variant. Persist the affine as `transform.json` and keep the VALIS object/handle for the non-rigid field.

!!! note "Determinism"
    Pin feature-detector and RANSAC seeds where the backend exposes them; record the VALIS version in the transform metadata so warps are reproducible.

## Mask & outline extraction

Per scan, at a low pyramid level:

1. RGB → HSV; take the **S** channel; Gaussian blur to suppress tile texture.
2. **Otsu** threshold (×~0.7 for HE) → binary tissue.
3. Morphology: `closing(disk(10))` → `remove_small_objects` → `dilation` → `binary_fill_holes`.
4. Contours (`cv2.findContours` or `shapely`) → simplify → scale vertices to level-0 → polygon array. Multiple disjoint regions → list of polygons.
5. **Cross-stain intersection**: `shapely` intersection of all stains' `elastic` outlines for the scan.

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

Registered OME-TIFFs + transforms, per-variant per-stain outlines (polygon arrays + GeoJSON export), the per-scan cross-stain intersection, and the per-scan biopsy axis.
