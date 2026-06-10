# Project context

Histology training & evaluation pipeline for MIL models on whole-slide images.

Design docs in `docs/design/` are the **authoritative source of truth**.
Always read the relevant design doc before suggesting implementation changes.
Keep docs at overview altitude for now — exact schemas (bundle, manifest) are deliberately deferred.

## Core entities

Dataset → Patient → Biopsy → Scan (one per stain). Labels keyed per biopsy; **optional** (eval-only datasets may have none). Bag = patch embeddings for one model input instance.

Stains: H&E + IHC stains. Some scores arrive as 4 per-biopsy quartiles — quartile is **carried metadata, not a spatial index** (regions unknown); averaged in preprocessing.

## Source variants (critical concept)

Each scan exists as `raw` / `rigid` / `elastic`.
- **Training and all metrics use `raw` only.**
- **Registration affects heatmaps only**, never the reported numbers. Elastic distortion changes visuals, not scores.
- `source_variant` is a first-class bag identifier field, not folded into patch config.

## Cohorts vs. splits (terminology — avoid overloading "test")

- **Cohort** = patient-level partition of a **patient set**: `development` (all CV happens here) and `holdout` (few locked-away patients, evaluated once at the end / "lockbox").
- **Split** = fold-level roles inside development, assigned by seeds: `train` / `val` / `test`.
- Rule: train/val/test ONLY describe folds in development; reserved patients are ONLY ever "holdout", never "test". → "CV test score" vs "holdout score" are distinct, unambiguous.
- There are only TWO cohorts. The old "full" bundle = development ∪ holdout ("all"), materialized only for final production retrain after holdout is consumed — not a third cohort.
- **Patient set** is a named, possibly multi-dataset collection of patients; splits and bundles both derive from it. Splits are over the patient set, so different stain/embedding bundles share aligned folds (a patient lacking a stain just contributes no bag but keeps its fold).
- **Bundle ↔ cohort**: a bundle materializes a patient set for one (stain · embedding · variant · patch config) and contains EVERY bag, each tagged with its `cohort`. Cohort is a manifest column, NOT a separate bundle. Stages pick a **cohort scope enum** (`development` / `holdout` / `all`) — never a list. The union = `all`. This is the resolved answer to "train/eval on a list of cohorts" (rejected as unintuitive/leak-prone).
- Wired into configs: `docs/configs/patient_sets.yaml` (members + holdout designation → frozen hashed membership), `seeds.yaml` references `patient_set:`, preprocessing `bundle:` block assembles from a patient set, `training.yaml`/`evaluation.yaml` carry `cohort:`. `patient_exclusion` is gone (lifted up to the patient set's holdout).

## Metadata flow (decided)

The scan manifest (Stage 1 ingestion, IDs → WSI path) is the pipeline contract. Metadata rides on it as columns **at any entity level** (patient/biopsy/scan — coarser levels just repeat across rows); no separate metadata file. Preprocessing **forwards** this metadata into the bundle manifest and into each BEAM `/metadata`, so it's available to reports/heatmaps without re-joining the source. (Resolves the old "metadata file scope" open question.)

## Settled decisions

- Five stages, run individually, connected by Snakemake (Stage 1 ingestion is outside Snakemake — user-written bridge).
- Folds are **NOT** generated in preprocessing — they belong to Model Training, to support the seed sweep (independent model-seed × fold-seed variation).
- **No fitted statistics in bundles** — bundles carry raw labels/embeddings; normalization etc. is computed at training time from the train split only. This is what makes the 3-bundle patient-exclusion scheme leakage-free.
- Patient exclusion produces 3 bundles: training (minus test set), full, held-out (test only).
- Embedding reuse via **content-addressed cache** (key: coords + patch size + resolution + embedding model + source variant).
- Pipeline reads a **manifest** (IDs → paths), never the on-disk layout — so folder vs flat ingestion both work.

## Storage formats

Binary is the source of truth; GeoJSON is the TissUUmaps view (pipeline never computes from GeoJSON).
- Embeddings → HDF5. Patch coordinates → HDF5. Tissue outlines → **polygon arrays** (+ GeoJSON export). Transform matrices → JSON.
- **BEAM** = the project's own per-biopsy, per-model evaluation result format. Stored as HDF5, `{biopsy_id}__{model_id}.beam.h5`. Contains attention (raw/sigmoid/rank), prediction, labels, patch coords, outline (polygon array, divisible into quartiles), metadata. Appendable by design. (Name is provisional — easy to rename.)
- Detailed format specs live in `docs/formats/` (beam, embeddings-and-patches, outlines) — kept separate so design docs stay overview-level. Stage docs give rough bullets + a link.

## Reports & experiments

- Reports = regenerable **view** over manifests/`runs.parquet`/BEAM; static HTML, **Plotly** interactive plots, standalone `report_assets/reports.css` (no MkDocs dep), hybrid layout (self-contained run folders + index files).
- **Runs are generated, not configured.** `training.yaml` is an **experiment definition** (umbrella `experiment` name + matrix) that fans out into many runs (seed sweep × HPO). Config count = O(experiments), not O(models). Each run → a run record → aggregated into `results/runs.parquet` (the master tagged, exportable table). Reports index by **tags** (faceted: experiment/bundle/architecture/stain), never by name-parsing.
- **HPO is segregated**: under `results/experiments/{exp}/hpo/` with its own index; `keep_checkpoints: all|top_n|none` (default top_n). Seed-sweep models live under `sweep/` (the keepers). Workflow: HPO explores → promote best N → seed sweep on them.
- **Table export, two levels**: client-side CSV/JSON buttons + link to canonical backing artifact (CSV/Parquet) at a stable path for the user's own Python scripts.
- High-value pre-model reports (independent of training, build first): fold×cohort composition, dataset distribution, cohort composition.
- Design: `docs/design/11-reports.md`; config: `docs/configs/reports.yaml`.

## Configuration

- **`base.yaml`** holds shared roots (raw/normalized/processed/bundles/results) + registry paths + rarely-edited settings; layered UNDER every stage config (base first, stage overrides win). Changing an output path = one edit.
Draft per-stage Snakemake configs in `docs/configs/` (wsi_transformation, preprocessing, training, evaluation, heatmaps) + `seeds.yaml` split registry. All marked DRAFT. Each has a wrapper md page (`docs/configs/*.md`) that embeds the YAML via pymdownx.snippets; listed under a collapsible Configuration nav menu. Overview in `docs/design/10-configuration.md`. Per-file (not monolithic) because stages run individually; single merged config left as an open alternative.
- **`seeds.yaml` is a project-wide named registry**: top-level `seed_sets` maps names → split configs; a run picks one via `seed_set:`. Models sharing a name get identical splits.
- **Label balancing** lives in `training.yaml` (per fold, train split only).
- **Augmentation lives in PREPROCESSING, not training** — histology augmentation (flips/rotations/stain jitter) changes pixels, so it runs the embedding foundation model on augmented patches and caches each variant as its own embedding set (augmentation_id is part of the cache key). Training only picks whether to sample them (`use_augmented_embeddings`).

## Six stages

1. Data Ingestion (outside Snakemake) — normalize raw → standard format + per-biopsy label CSV
2. WSI Transformation — registration (raw/rigid/elastic) + tissue outlines + cross-stain intersection
3. Dataset Preprocessing — label processing, patch generation, patch embedding (+cache), bundle prep
4. Model Training — fold generation (split registry `seeds.yaml`), seed sweep, HPO, training function
5. Evaluation — inference, per-biopsy aggregation into BEAM files, reports
6. Heatmap Generation — attention overlays (PNG) + TissUUmaps GeoJSON. NOTE: attention is in the raw frame, so coords must be pushed through the variant's transform matrix to overlay on rigid/elastic.

## Doc structure

Design: `01-overview` · `02-data-model` · `03-data-ingestion` · `04-wsi-transformation` · `05-dataset-preprocessing` · `06-model-training` · `07-evaluation` · `08-heatmaps` · `09-open-questions` · `10-configuration`
Formats: `docs/formats/{beam,embeddings-and-patches,outlines}.md`
Configs: `docs/configs/*.yaml` (drafts)
