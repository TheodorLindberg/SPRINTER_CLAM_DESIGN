# Stage 1 · Data Ingestion

Normalizes a raw dataset into the standardized format the rest of the pipeline consumes.

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

---

## Labels

Alongside the scans, a CSV holds per-biopsy labels keyed by patient, biopsy, and stain:

- Proliferation / expression scores **per quartile** (4 per biopsy). Region information is unavailable, so these are averaged in a later step.
- Gleason grade.
- Biopsy length and tumor length.

Labels are optional — an evaluation-only dataset may ship without them.

---

## Bag naming inputs

Ingestion establishes the identifiers a bag is later built from: dataset origin, patient index, biopsy index, and staining method. The remaining components (patching configuration, source variant, embedding model) are assigned downstream. See [Data Model · Bag naming](02-data-model.md#bag-naming).

---

## Metadata note

A metadata file that survives through the pipeline would let later stages reach extra information (e.g. stain details). Scope is undecided — metadata can be per-patient, per-biopsy, or per-scan. The recommended minimal approach (extra columns on the existing entity manifests, resolved by ID inheritance) is described in [Open Questions](09-open-questions.md#metadata-file-scope).

---

## Open items

- Define the exact pre–preprocessing normalized structure (the manifest contract).
