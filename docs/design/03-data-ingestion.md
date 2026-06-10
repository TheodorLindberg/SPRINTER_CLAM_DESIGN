# Stage 1 · Data Ingestion

Normalizes a raw dataset into the standardized format the rest of the pipeline consumes.

> **In** raw dataset files (WSIs + source labels) · **Out** normalized scans, a [scan manifest](#scan-manifest-the-contract), a per-biopsy label CSV

*Go deeper: [Specification — input contract](../spec/data-ingestion.md) (manifest + label schemas, invariants, acceptance criteria).*

This stage sits **outside Snakemake**: every dataset arrives differently, so each user writes their own bridge to produce the normalized structure. We supply a ready-made ingester only for our own input dataset; others implement to the same contract.

---

## Normalized structure

Ingestion produces two things: the scan files and a **scan manifest**. The manifest is the pipeline's only interface — **the on-disk layout is free**, because no stage parses the directory structure.

A folder hierarchy mirroring the entity hierarchy is the suggested convention:

```text
patient_<x>/
  biopsy_<x>/
    <stain>.<ext>      # <ext> = any OpenSlide-supported format
```

A flat layout works equally well; folders vs. flat is the ingester's choice.

## Scan manifest (the contract)

One row per scan — the entity ids, the WSI path, and any metadata columns:

| Column | Role |
|---|---|
| `dataset_id` | dataset source (+ optional version) |
| `patient_id` | unique within the dataset |
| `biopsy_id` | unique within the patient |
| `stain` | `HE` / `Ki67` / `PSA` |
| `wsi_path` | path to the scan file |
| *(any others)* | metadata, at any level — see below |

A scan is identified by **`(biopsy_id, stain)`** — a biopsy has at most one scan per stain, so there is **no separate `scan_id`**. Where a single-token handle is convenient (paths, filenames) it is just `{biopsy_id}__{stain}`.

### Metadata, at any level

Extra columns describe the patient, biopsy, or scan (age, site, scanner, stain details, …). A column's *level* is simply which entity it describes — patient-level values repeat across that patient's rows; no separate metadata file is needed. This metadata is **carried through preprocessing and forwarded into the bundle and the [BEAM](../formats/beam.md) file**, so it reaches reports and heatmaps without re-joining the source. See the [metadata decision](09-open-questions.md#metadata-file-scope).

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
