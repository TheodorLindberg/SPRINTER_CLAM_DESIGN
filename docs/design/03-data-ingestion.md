# Stage 1 · Data Ingestion

Every dataset we receive is organised differently — different folder layouts, file names, and label spreadsheets. This stage repackages one such dataset into the single predictable shape the rest of the pipeline reads, so nothing further down has to care where the original files came from. It is the one stage you write yourself: we ship a ready-made ingester for our own dataset, and anyone adding a new dataset writes a small bridge script that produces the same shape.

> **In** a raw dataset (WSIs + source labels) · **Out** normalized scans, a [scan manifest](#the-scan-manifest), and a per-biopsy label table

*Go deeper: [Specification — input contract](../spec/data-ingestion.md) (full manifest and label schemas, invariants, acceptance criteria).*

This stage sits **outside Snakemake**. Because every dataset arrives differently, there is no single ingester we can ship for all of them — but what every bridge must produce is identical, and is fixed by the [input contract](../spec/data-ingestion.md).

---

## What ingestion produces

Two things:

1. **The scan files** — the whole-slide images, in any OpenSlide-supported format.
2. **A scan manifest** — a small file that lists every scan and where to find it.

Optionally, it also produces one or more **label tables** (see [Labels](#labels) below).

### File layout is free

The pipeline finds scans through the manifest, never by walking the directory tree, so the on-disk layout is up to you. A folder hierarchy that mirrors the entities is a convenient default:

```text
patient_<x>/
  biopsy_<x>/
    <stain>.<ext>        # any OpenSlide-supported format
```

A flat folder, symlinks, or any other arrangement works equally well, as long as the paths in the manifest resolve.

## The scan manifest

The manifest is the **contract** between your ingester and the rest of the pipeline: a single file that maps each scan's identity to its file path. Every later stage resolves scans through it.

It is a hierarchical document — dataset → patients → biopsies → scans — written as YAML or JSON. A scan is identified by its biopsy and its stain (`HE`, `Ki67`, `PSA`, …); there is at most one scan per stain per biopsy, so scans need no separate id of their own.

Metadata (a patient's age, a biopsy's grade, a scanner model, …) can be attached at any level and travels with the data downstream — see [Metadata](#metadata) below.

The exact structure, with a worked example, lives in the **[input-contract spec](../spec/data-ingestion.md#scan-manifest-yaml-json)**.

### Metadata

A `metadata:` block can sit at the **dataset, patient, biopsy, or scan** level, so each value is written once at the level it describes. Each value keeps its level all the way through the pipeline: preprocessing forwards it into the bundle and into each [BEAM](../formats/beam.md) result file, grouped by level, so reports and heatmaps always know whether a value describes the patient, the biopsy, or the scan — without re-joining the original source. See the [metadata decision](09-open-questions.md#metadata-file-scope).

---

## Labels

Labels are the values a model is trained to predict (a Gleason grade, a Ki67 score). They are kept **separate from the manifest**, because they are prediction targets rather than descriptive metadata, and they often arrive later and in different shapes.

Each label table covers one label family, keyed by patient and biopsy, so families with different shapes can coexist. For this project:

- **Proliferation / differentiation scores**, as four per-biopsy quartiles. The region each quartile came from is unknown, so they are averaged into a single biopsy value in a later step.
- **Gleason grade.**
- **Biopsy and tumor length.**

Preprocessing merges the configured tables into one normalized label set. Labels are **optional**: an evaluation-only dataset may ship without any. See the [input-contract spec](../spec/data-ingestion.md#label-tables-optional).

---

## What this stage sets up downstream

Ingestion fixes the identifiers a model input ([bag](02-data-model.md#entity-definitions)) is later built from: which dataset, patient, biopsy, and stain it came from. The remaining pieces — patching configuration, source variant, embedding model — are assigned by later stages. See [Data Model · Bag naming](02-data-model.md#bag-naming).
