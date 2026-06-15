# Stage 2 · WSI Transformation

Each biopsy is scanned once per stain, and the scanner places every stain on the slide at a slightly different angle and position — so the same piece of tissue does not line up between, say, the H&E and the Ki67 image. This stage does two things: it lines the stains up with each other (**registration**), and it traces the outline of the tissue on each slide so later stages know where to look.

> **In** normalized scans (via the manifest) · **Out** raw / rigid / elastic variants + transformation matrices, and tissue outlines

*Go deeper: [Specification](../spec/wsi-transformation.md) · [Implementation](../impl/wsi-transformation.md).*

It is its own stage because registration is expensive, and its outputs are reused by every downstream patch and embedding configuration.

---

## Source variants

Each scan is produced in up to three variants:

| Variant | What it is | Alignment | Tissue fidelity |
|---|---|---|---|
| **Raw** | Original scans | None | Exact |
| **Rigid** | Registered TIFF + transformation matrix; rotation/translation only | Coarse | Exact (rotational interpolation noise is negligible) |
| **Elastic** | Biopsy stretched so IHC stains align tightly to H&E | Tight | Locally distorted |

A rigid heatmap is effectively equivalent to a raw one, just coarsely aligned across stains. Stains align to the **reference stain** — H&E by default, configurable via `reference_stain` in [`base.yaml`](../configs/base.md).

---

## Tissue outlines

Outlines are produced as part of the registration step, not as a separate masking stage. The tissue source is configurable: by default the segmentation built into VALIS (the registration toolkit — it reuses the very masks it registers on), or an `hsv_otsu` mask. Outlines are stored as polygon vertex arrays (the pipeline's source of truth), with a GeoJSON export for TissUUmaps viewing. → [Outlines spec](../formats/outlines.md).

!!! note "Comparing tissue methods (development)"
    Both methods can be emitted to method-tagged paths and overlaid in the QC PNG, so they can be compared early on; only the configured method feeds downstream stages.

- **Raw outline** — per stain.
- **Rigid-registered outline** — per stain.
- **Elastic-registered outline** — per stain.
- **Cross-stain intersection outline** — the region where tissue is present in *every* stain. It comes only from the elastic registration (the one alignment tight enough to overlap stains meaningfully) and is saved per scan rather than per patient, to keep things simple.

---

## Training vs. heatmap roles

This stage exists almost entirely to serve heatmaps. The split in responsibility is deliberate:

- **Training and all performance metrics use raw scans.** Registration never touches the numbers.
- **Registration only affects heatmap generation**, offering a three-way choice based on how much cross-stain alignment a visual needs:
    - **Raw** — no alignment.
    - **Rigid** — coarsely aligned, artifact-free.
    - **Elastic** — tightly aligned for overlay, with tissue distortion.

!!! note "Why distortion is safe here"
    Because a raw-trained model is applied to registered images only for heatmaps (never for metrics), patch-content distortion under elastic registration affects visuals but never the reported numbers.

---

## Outputs

All from the single registration step:

- Registered TIFFs (rigid, elastic) plus transformation matrices.
- Per-stain outlines for each variant (polygon arrays + GeoJSON), and the per-scan cross-stain intersection.
- A **low-res QC PNG** per scan — the tissue outline drawn on the tissue — plus an HE thumbnail overlaying all stains' outlines, to verify results at a glance.
