# BEAM format

**BEAM** — *Biopsy Evaluation & Attention Map* — is the project's own per-biopsy, per-model evaluation result, stored as **HDF5**:

```text
{biopsy_id}__{model_id}.beam.h5
```

Produced by [Stage 5 · Evaluation](../design/07-evaluation.md); consumed by reports and [Stage 6 · Heatmaps](../design/08-heatmaps.md). HDF5 is chosen because it is **appendable** — enrichment steps can add datasets or groups without breaking existing readers.

!!! tip "Provisional name"
    The format name can be renamed with a single find-replace.

---

## Layout

```text
{biopsy_id}__{model_id}.beam.h5
│
├─ ⚙ attributes ··········· provenance — see table below
│
├─ patches/
│  ├─ coords ·············· (N, 2|4) int   x, y (, w, h) · WSI frame
│  └─ size ················ patch size / resolution
│
├─ attention/
│  ├─ raw ················· (N,) float
│  ├─ sigmoid ············· (N,) float
│  └─ rank ················ (N,) float
│
├─ outline/ ··············· polygon arrays — identical to the Outlines spec
│  ├─ polygon ············· (M, 2) float
│  └─ quartiles/ ·········· optional · q0 … q3, each (Mᵢ, 2) float
│
├─ prediction ············· prediction per model
├─ labels/ ················ true labels where available · name → value (+ type)
├─ embeddings ············· (N, D) float · optional, else referenced from bundle
└─ metadata/ ·············· model info, registration info, free-form
```

---

## Root attributes

| Attribute | Meaning |
|---|---|
| `format_version` | BEAM schema version |
| `biopsy_id`, `patient_id`, `dataset_id` | Entity provenance |
| `model_id`, `embedding_model_id` | Which model + embedder produced this |
| `evaluation_tag` | Readable evaluation-run name |
| `stain` | Stain of the evaluated scan(s) |
| `source_variant` | `raw` / `rigid` / `elastic` |
| `patch_config_id`, `patch_size`, `patch_resolution` | Patching configuration |
| `quartile` | Carried metadata — **not** a spatial index |

---

## Outline

The tissue outline used follows the **exact same layout as the [Outlines spec](outlines.md#polygon-array)** — a `polygon` vertex array, optionally divided into `quartiles/q0 … q3`. GeoJSON is produced only as a separate viewing export, never stored inside BEAM.

---

## Field mapping

| Evaluation field | Location |
|---|---|
| Attention (raw, sigmoid, rank) | `attention/{raw,sigmoid,rank}` |
| Prediction per model | `prediction` |
| True labels (where available) | `labels/` |
| Patches | `patches/coords` |
| Patch size | attr `patch_size` + `patches/size` |
| Tissue outline used | `outline/` (polygon array) |
| Quartile | attr `quartile`; `outline/quartiles` when subdivided |
| Stain | attr `stain` |
| Patient | attr `patient_id` |
| Model info | attr `model_id` + `metadata/` |
| Embedding model | attr `embedding_model_id` |
| Registration / source variant | attr `source_variant` + `metadata/` |
| Metadata | `metadata/` |
