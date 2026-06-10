# Spec · Data Ingestion (input contract)

The contract a [Stage 1](../design/03-data-ingestion.md) ingester must satisfy so the Snakemake pipeline (Stage 2 onward) can run. Stage 1 is user-written and outside Snakemake; **this page is the boundary between the two**. [Overview](../design/03-data-ingestion.md) · **Specification**.

## Normalization contract

Per `dataset_id`, an ingester produces:

- the **scan files** (any OpenSlide-supported format), in **any on-disk layout**;
- a **scan manifest** — the load-bearing interface (the pipeline reads this, never the directory layout);
- an optional **label CSV**.

"Normalized" means *conforms to the manifest + label schemas below* — not a fixed folder structure.

## Scan manifest

One row per scan.

| Column | Type | Required | Notes |
|---|---|---|---|
| `dataset_id` | string | yes | source (+ optional version), e.g. `sahlgrenska_2018` |
| `patient_id` | string | yes | arbitrary **de-identified** index (e.g. `p0001`) — never a real PAD / MRN / scan number |
| `biopsy_id` | string | yes | unique within the patient |
| `stain` | enum | yes | the dataset's normalized vocabulary — `HE` / `Ki67` / `PSA` here |
| `wsi_path` | string | yes | resolvable path to the scan file |
| *(any others)* | any | no | metadata at patient / biopsy / scan level — forwarded downstream |

- **Scan key:** `(biopsy_id, stain)` within a patient; `(dataset_id, patient_id, biopsy_id, stain)` globally. No `scan_id` column — it is [derived](../design/02-data-model.md#identifiers).
- Stain names are normalized to the fixed vocabulary (no `H&E` vs `HE` drift).

## Label CSV (optional)

Keyed by `(dataset_id, patient_id, biopsy_id)`; columns are the **raw** dataset scores (e.g. Ki67 / PSA quartiles, Gleason grade, biopsy / tumor length). Entirely absent for label-free (evaluation-only) datasets. Derived labels are computed later in [preprocessing](preprocessing.md).

## Invariants

- Every `wsi_path` exists and is OpenSlide-readable.
- `(dataset_id, patient_id, biopsy_id, stain)` is **unique** — no duplicate scans.
- Each `stain` is in the dataset's normalized vocabulary.
- If registration is used, every biopsy has an **`HE`** scan (HE is the registration reference frame).
- Label rows join to scans on the **full** `(dataset_id, patient_id, biopsy_id)` key; rows with no matching biopsy are an error.
- `patient_id` carries no real-world identity.

## Acceptance criteria

- A manifest validator passes a well-formed dataset and **fails** on: a missing/unreadable `wsi_path`; a duplicate `(patient, biopsy, stain)`; an unknown stain; a label row with no matching biopsy; a biopsy missing `HE` when registration is enabled.
- The same dataset passes whether stored as a folder hierarchy or flat, as long as the manifest resolves.
