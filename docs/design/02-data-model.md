# Data Model

## Entity hierarchy

```mermaid
erDiagram
    DATASET ||--o{ PATIENT : contains
    PATIENT ||--o{ BIOPSY : has
    BIOPSY ||--o{ SCAN : has
    SCAN }o--|| STAIN : uses
    SCAN ||--o{ PATCH : tiled_into
    PATCH ||--o{ EMBEDDING : encoded_as
    BIOPSY ||--o{ BAG : forms
    BIOPSY ||--o{ LABEL : has
```

## Entity definitions

| Entity | Definition |
|---|---|
| **Dataset** | A collection of patients, biopsies, scans, and labels from one source. Has an explicit version. |
| **Patient** | A biological individual. The unit reserved or excluded for held-out evaluation. |
| **Biopsy** | A tissue sample from one patient. The unit labels and bags are keyed on. |
| **Scan** | A digitized WSI from one biopsy and one stain. Any OpenSlide-supported format. |
| **Stain** | The staining method applied to a scan (e.g. H&E and IHC stains). |
| **Patch** | A crop extracted from a scan at a configured size and resolution. |
| **Embedding** | A feature vector produced from a patch by an embedding model. |
| **Bag** | The set of patch embeddings used as one model input instance. |
| **Label** | A target value attached to a biopsy. Has a name, type, and value. Optional. |

---

## Source variants

A scan exists in up to three **source variants**, produced in [WSI Transformation](04-wsi-transformation.md):

- **`raw`** — original scan; misaligned across stains. **All training and metrics use this.**
- **`rigid`** — rotation/translation only; coarsely aligned, artifact-free.
- **`elastic`** — tightly aligned to H&E for overlays, at the cost of local tissue distortion.

The source variant is a first-class identifier component, not folded into the patching configuration, because it changes the pixel content patches are drawn from.

---

## Identifiers

All identifiers are stable and recorded in manifests, never inferred from filenames alone.

| Identifier | Notes |
|---|---|
| `dataset_id` | Explicit version, never `latest` |
| `patient_id` | Unique within dataset |
| `biopsy_id` | Unique within patient |
| `scan_id` | One per (biopsy, stain) |
| `patch_config_id` | Patch size, resolution, overlap/stride |
| `source_variant` | `raw` / `rigid` / `elastic` |
| `embedding_model_id` | Model name + version |
| `bag_id` | Fully qualified — see below |

### Bag naming

A bag is identified by dataset origin, patient, biopsy, stain, patching configuration, source variant, and embedding model:

```
bag_id = {dataset_id}__p{patient}__b{biopsy}__s{stain}__patch-{patch_config_id}__src-{source_variant}__emb-{embedding_model_id}
```

Source variant is kept as its own field so that raw / rigid / elastic bags for the same biopsy are distinct and traceable.

---

## Storage formats

Binary is the source of truth; GeoJSON is the view. Anything meant for interactive inspection in TissUUmaps is also exported as GeoJSON, but the pipeline never depends on the GeoJSON for computation.

Detailed layouts live in the [format specs](../formats/beam.md); this is the overview.

| Artifact | Format | Spec |
|---|---|---|
| Embeddings | **HDF5** (binary, per scan) | [Embeddings & patches](../formats/embeddings-and-patches.md) |
| Patch coordinates | **HDF5** (binary arrays) | [Embeddings & patches](../formats/embeddings-and-patches.md) |
| Tissue outlines | **Polygon arrays** (+ GeoJSON export) | [Outlines](../formats/outlines.md) |
| Patch / tissue geometry for viewing | **GeoJSON** | [Outlines](../formats/outlines.md) |
| Transformation matrices | JSON | — |
| Evaluation results | **BEAM** (HDF5) | [BEAM](../formats/beam.md) |
| Heatmaps | PNG + GeoJSON | [Heatmaps](08-heatmaps.md) |

---

## Label model

Labels are keyed per biopsy (by patient, biopsy, stain). Each label carries:

- **Name** — identifier.
- **Type** — binary, continuous, categorical, etc.
- **Value**.

Raw labels come from the ingestion CSV; **derived labels** (averages, max, binary thresholds, …) are computed in [Dataset Preprocessing](05-dataset-preprocessing.md). The set of derived labels is intentionally extendable per dataset.

!!! note "Labels are optional"
    A bundle prepared for evaluation-only on an external dataset may carry no labels at all. Every downstream stage must tolerate their absence.

!!! note "Quartiles are metadata, not geometry"
    Some scores (e.g. proliferation/expression indices) arrive as four per-biopsy quartile values. Because region information is unavailable, the quartile is carried as metadata only — it is **not** a spatial index — and the values are averaged in preprocessing.
