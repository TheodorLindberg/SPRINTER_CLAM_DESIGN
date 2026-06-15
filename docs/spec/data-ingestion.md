# Spec · Data Ingestion (input contract)

The contract a [Stage 1](../design/03-data-ingestion.md) ingester must satisfy so the Snakemake pipeline (Stage 2 onward) can run. Stage 1 is user-written and outside Snakemake; **this page is the boundary between the two**. [Overview](../design/03-data-ingestion.md) · **Specification**.

## Normalization contract

Per `dataset_id`, an ingester produces:

- the **scan files** (any OpenSlide-supported format), in **any on-disk layout**;
- a **scan manifest** (YAML or JSON) — the load-bearing interface (the pipeline reads this, never the directory layout);
- an optional **label table**.

"Normalized" means *conforms to the manifest + label schemas below* — not a fixed folder structure.

## Scan manifest (YAML / JSON)

A hierarchical document — `dataset → patients → biopsies → scans` — with an optional `metadata:` block at **any level**:

```yaml
dataset_id: sahlgrenska_2018
metadata: { scanner: Hamamatsu, year: 2018 }        # dataset-level (optional)
patients:
  p0001:                                            # an index within the dataset
    metadata: { age: 67, site: gothenburg }         # patient-level
    biopsies:
      b01:
        metadata: { gleason: 7 }                    # biopsy-level
        scans:
          HE:   { path: data/.../p0001_b01_HE.ndpi }
          Ki67: { path: data/.../p0001_b01_Ki67.ndpi, metadata: { antibody: MIB-1 } }   # scan-level
          PSA:  { path: data/.../p0001_b01_PSA.ndpi }
```

- **Required:** `dataset_id`; each patient → biopsy → scan keyed by `stain` with a `path`. Stains use the normalized vocabulary (`HE` / `Ki67` / `PSA`) — no `H&E` vs `HE` drift.
- A scan is `(patient, biopsy, stain)`; there is **no `scan_id` field** (it is [derived](../design/02-data-model.md#identifiers)).
- `metadata:` is optional at the **dataset, patient, biopsy, and scan** levels. Each entry **keeps its level** all the way downstream: preprocessing forwards it into the bundle and into the [BEAM](../formats/beam.md) `/metadata`, grouped by level — so a reader always knows whether a value describes the patient, biopsy, or scan.

!!! note "Tabular vs hierarchical"
    The *input* manifest is hierarchical → YAML/JSON. The *generated* per-row manifests (bag manifest, folds) stay tabular (CSV/Parquet) — they are row data, not nested config.

## Label tables (optional)

Labels are supplied as **one or more tables** (CSV/Parquet), **separate from the manifest** — they are training **targets**, not descriptive metadata, and typically arrive later and in different shapes. Each table holds **one label family**, keyed by `(dataset_id, patient_id, biopsy_id)`:

- `ki67.csv` → Ki67 quartile columns
- `gleason.csv` → an ordinal grade
- `lengths.csv` → biopsy / tumor length

[Preprocessing](preprocessing.md) reads the configured sources, **merges them on the biopsy key**, and normalizes them into one **long** label table — `(dataset, patient, biopsy, label_name, label_value, label_type)`, one row per biopsy × label — which absorbs any mix of types.

- **Why separate files:** each family keeps its natural schema; a new label set is a new file (no editing existing data); a biopsy missing from a file is simply unlabelled for that family.
- Absent entirely for label-free (evaluation-only) datasets.

## Invariants

- Every scan `path` exists and is OpenSlide-readable.
- A scan `(patient, biopsy, stain)` is unique by construction (the hierarchy can't express duplicates).
- Each `stain` is in the dataset's normalized vocabulary.
- If registration is used, every biopsy has a scan for the **`reference_stain`** (default `HE`, from `base.yaml`) — it defines the registration reference frame.
- Label rows join to scans on the **full** `(dataset_id, patient_id, biopsy_id)` key; rows with no matching biopsy are an error.

## Acceptance criteria

- A manifest validator passes a well-formed dataset and **fails** on: a missing/unreadable scan `path`; an unknown stain; a label row with no matching biopsy; a biopsy missing `HE` when registration is enabled.
- The same dataset passes whether stored as a folder hierarchy or flat, as long as the manifest resolves.
- Every `metadata:` value reaches the BEAM `/metadata` tagged with its level.
