# Stage 2 · WSI Transformation

Raw scans carry arbitrary per-stain rotation and translation, so tissue does not align across stains. This stage corrects that through registration, and detects the tissue outlines patch generation will use.

> **In** normalized scans (via the manifest) · **Out** raw / rigid / elastic variants + transformation matrices, and tissue outlines

*Go deeper: [Specification](../spec/wsi-transformation.md) · [Implementation](../impl/wsi-transformation.md).*

It is kept as its own stage because registration is expensive and its outputs are reused across every downstream patch and embedding configuration.

---

## Source variants

Each scan is produced in up to three variants:

| Variant | What it is | Alignment | Tissue fidelity |
|---|---|---|---|
| **Raw** | Original scans | None | Exact |
| **Rigid** | Registered TIFF + transformation matrix; rotation/translation only | Coarse | Exact (rotational interpolation noise is negligible) |
| **Elastic** | Biopsy stretched so IHC stains align tightly to H&E | Tight | Locally distorted |

A rigid heatmap is effectively equivalent to a raw one, just coarsely aligned across stains.

---

## Tissue outlines

Outlines are produced **in the registration step**, from a configurable tissue source — **VALIS's own segmentation** (default; the same masks it registers on) or an `hsv_otsu` mask — not a separate masking stage. They are stored as **polygon vertex arrays** (the pipeline's source of truth), with a **GeoJSON** export for TissUUmaps viewing. → [Outlines spec](../formats/outlines.md).

!!! note "Comparing tissue methods (development)"
    Both methods can be emitted to method-tagged paths and overlaid in the QC PNG, so they can be compared early on; only the configured method feeds downstream stages.

- **Raw outline** — per stain.
- **Rigid-registered outline** — per stain.
- **Elastic-registered outline** — per stain.
- **Cross-stain intersection outline** — the region with tissue present in *every* stain. It comes only from the overlapping elastic registration and is saved **per scan** (rather than per patient) to keep complexity down.

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
