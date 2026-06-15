# Stage 1 · Data Ingestion

Normalizes a raw dataset into the standardized format the rest of the pipeline consumes.

> **In** raw dataset files (WSIs + source labels) · **Out** normalized scans, a [scan manifest](#scan-manifest-the-contract), a per-biopsy label CSV

*Go deeper: [Specification — input contract](../spec/data-ingestion.md) (manifest + label schemas, invariants, acceptance criteria).*

This stage sits **outside Snakemake**: every dataset arrives differently, so each user writes their own bridge to produce the normalized structure. We supply a ready-made ingester only for our own input dataset; others implement to the same contract.

---

## Normalized structure

Ingestion produces the **scan files** and a **scan manifest**.

The manifest maps each scan's ids → file path; downstream stages resolve scans through it and never read the directory tree. The on-disk layout is therefore unconstrained — a folder hierarchy mirroring the entities is a convenient default:

```text
patient_<x>/
  biopsy_<x>/
    <stain>.<ext>      # <ext> = any OpenSlide-supported format
```

Any other layout (flat, symlinked) works as long as the manifest's paths resolve.

## Scan manifest (the contract)

A hierarchical **YAML/JSON** document — `dataset → patients → biopsies → scans` — with an optional `metadata:` block at any level:

```yaml
dataset_id: sahlgrenska_2018
patients:
  p0001:
    metadata: { age: 67 }                     # patient-level
    biopsies:
      b01:
        metadata: { gleason: 7 }              # biopsy-level
        scans:
          HE:   { path: .../p0001_b01_HE.ndpi }
          Ki67: { path: .../p0001_b01_Ki67.ndpi, metadata: { antibody: MIB-1 } }
```

A scan is identified by **`(biopsy_id, stain)`** — at most one scan per stain per biopsy, so there is **no separate `scan_id`** (a `{biopsy_id}__{stain}` handle is derived where needed). Full schema, invariants, and acceptance criteria: **[input-contract spec](../spec/data-ingestion.md)**.

### Metadata, at any level

`metadata:` can sit at the **dataset, patient, biopsy, or scan** level — nested, so a value is written once (no repetition). Each entry **keeps its level**: preprocessing forwards it into the bundle and the [BEAM](../formats/beam.md) `/metadata`, grouped by level, so reports and heatmaps know whether a value describes the patient, biopsy, or scan — without re-joining the source. See the [metadata decision](09-open-questions.md#metadata-file-scope).

---

## Labels

Labels are **separate from the manifest** (they are targets, not descriptive metadata) and supplied as **one or more tables** keyed by patient + biopsy — each table one label family, so different shapes coexist. For this project:

- Proliferation / differentiation scores **per quartile** (4 per biopsy). Region information is unavailable, so these are averaged in a later step.
- Gleason grade.
- Biopsy and tumor length.

Preprocessing merges the configured tables and normalizes them into one long label set. Labels are optional — an evaluation-only dataset may ship without any. See the [input-contract spec](../spec/data-ingestion.md#label-tables-optional).

---

## Bag naming inputs

Ingestion establishes the identifiers a bag is later built from: dataset origin, patient index, biopsy index, and staining method. The remaining components (patching configuration, source variant, embedding model) are assigned downstream. See [Data Model · Bag naming](02-data-model.md#bag-naming).

---

## Schema

The exact manifest structure, label tables, invariants, and acceptance criteria are the **[input-contract spec](../spec/data-ingestion.md)**. (Metadata is free-form nested `metadata:` per level — no reserved keys.)
