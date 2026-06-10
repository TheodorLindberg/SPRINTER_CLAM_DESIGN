# Stage 1 · Data Ingestion

Normalizes a raw dataset into the standardized format the rest of the pipeline consumes.

> **In** raw dataset files (WSIs + source labels) · **Out** normalized scans, a [scan manifest](#scan-manifest-the-contract), a per-biopsy label CSV

This stage sits **outside Snakemake**: every dataset arrives differently, so each user writes their own bridge to produce the normalized structure. We supply a ready-made ingester only for our own input dataset; others implement to the same contract.

---

## Normalized structure

A folder hierarchy is the suggested convention, mirroring the entity hierarchy:

```text
patient_<x>/
  biopsy_<x>/
    scan_<stain>.<ext>      # <ext> = any OpenSlide-supported format
```

A flat naming convention is also viable. **The pipeline does not depend on the on-disk layout** — it reads a generated manifest of IDs → paths (see [decision in Open Questions](09-open-questions.md#normalized-format-folders-vs-flat)). Folders vs. flat is therefore a presentation choice for the ingester, not a pipeline constraint.

## Scan manifest (the contract)

The manifest is the real interface to the pipeline: one row per scan, mapping entity ids → WSI path, plus any metadata columns.

- **Keys:** `dataset_id`, `patient_id`, `biopsy_id`, `stain`, `scan_id`, and the WSI path.
- **Metadata, at any level:** extra columns describe the patient, biopsy, or scan (e.g. age, site, scanner, stain details). A column's *level* is just which entity it describes — patient-level values simply repeat across that patient's rows. No separate metadata file is needed.

This metadata is **carried through preprocessing and forwarded into the bundle and the [BEAM](../formats/beam.md) file**, so it is available everywhere downstream (reports, heatmaps) without re-joining to the original source. See the [resolved metadata decision](09-open-questions.md#metadata-file-scope).

---

## Labels

Alongside the scans, a CSV holds per-biopsy labels keyed by patient, biopsy, and stain. The set is dataset-specific; for this project:

- Proliferation / expression scores **per quartile** (4 per biopsy). Region information is unavailable, so these are averaged in a later step.
- Gleason grade.
- Biopsy length and tumor length.

Labels are optional — an evaluation-only dataset may ship without them.

---

## Bag naming inputs

Ingestion establishes the identifiers a bag is later built from: dataset origin, patient index, biopsy index, and staining method. The remaining components (patching configuration, source variant, embedding model) are assigned downstream. See [Data Model · Bag naming](02-data-model.md#bag-naming).

---

## Open items

- Define the exact manifest column schema (keys + reserved metadata conventions).
