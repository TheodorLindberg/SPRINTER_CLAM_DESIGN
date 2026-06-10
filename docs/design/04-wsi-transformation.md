# Stage 2 · WSI Transformation

Raw scans carry arbitrary per-stain rotation and translation, so tissue does not align across stains. This stage corrects that through registration, and detects the tissue outlines patch generation will use.

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

Outlines are stored as JSON and GeoJSON (likely via VALIS):

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

- Registered TIFFs (rigid, elastic) plus transformation matrices.
- Per-stain outlines for each variant, in JSON and GeoJSON.
- Per-scan cross-stain intersection outline.
