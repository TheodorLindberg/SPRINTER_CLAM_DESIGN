# Project context

Histology training & evaluation pipeline for MIL models on whole-slide images.

Design docs in `docs/design/` are the **authoritative source of truth**.
Always read the relevant design doc before suggesting implementation changes.
Keep docs at overview altitude for now — exact schemas (bundle, manifest) are deliberately deferred.

## Git workflow

- **Commit often** — after each relatively large change, not as one big batch at the end.
- Keep messages **descriptive but short**: a one-line subject of what changed, plus a few bullets only when it genuinely helps.
- **Co-author every commit** with Claude Opus:
  ```
  Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>
  ```
- Commit on the working branch (`main`); don't push unless asked.

## Core entities

Dataset → Patient → Biopsy → Scan (one per stain). Labels keyed per biopsy; **optional** (eval-only datasets may have none). Bag = patch embeddings for one model input instance.
`dataset_id` names the dataset **source** (optionally versioned, e.g. `sahlgrenska_2018`) — NOT just a version. Patient = `(dataset_id, patient_id)` globally; this is what lets a cohort pool multiple sources. See `docs/design/12-appendix.md`.

Stains: H&E + IHC stains. Some scores arrive as 4 per-biopsy quartiles — quartile is **carried metadata, not a spatial index** (regions unknown); averaged in preprocessing.

## Source variants (critical concept)

Each scan exists as `raw` / `rigid` / `elastic`.
- **Training and all metrics use `raw` only.**
- **Registration affects heatmaps only**, never the reported numbers. Elastic distortion changes visuals, not scores.
- `source_variant` is a first-class bag identifier field, not folded into patch config.

## Cohort / role / split (terminology — avoid overloading "test")

- **Cohort** = a named, possibly multi-dataset **group of patients** (`cohorts.yaml`). "patient set" was COLLAPSED into this term.
- **Role** = per-patient tag within a cohort: `development` (all CV happens here) / `holdout` (few locked-away patients, evaluated once / "lockbox").
- **Split** = fold-level roles among development patients, assigned by seeds: `train` / `val` / `test`.
- Rule: train/val/test ONLY describe folds; reserved patients are ONLY ever `holdout`, never "test". → "CV test score" vs "holdout score" are distinct.
- Splits are over the cohort's development patients, so different stain/embedding bundles share aligned folds (a patient lacking a stain contributes no bag but keeps its fold). Membership is frozen + hashed; change → stale-splits warning.
- **Bundle = a prepared cohort**: materializes a cohort for one (stain · embedding · variant · patch config), EVERY patient present, each bag tagged with its `role`. Role is a manifest column, NOT a separate bundle. One cohort → many bundles. Stages pick a **`subset` enum** (`development` / `holdout` / `all`) — never a list. Union = `all`. (This is the resolved answer to "train/eval on a list of cohorts" — rejected as unintuitive/leak-prone.)
- Configs: `cohorts.yaml` (members + holdout designation → frozen hashed membership), `seeds.yaml` references `cohort:`, preprocessing `bundle:` block = prepared cohort, `model_experiment.yaml`/`evaluation.yaml` carry `subset:`. No `patient_exclusion` (lifted to the cohort's holdout role).

## Metadata flow (decided)

The scan manifest (Stage 1 ingestion, IDs → WSI path) is the pipeline contract. Metadata rides on it as columns **at any entity level** (patient/biopsy/scan — coarser levels just repeat across rows); no separate metadata file. Preprocessing **forwards** this metadata into the bundle manifest and into each BEAM `/metadata`, so it's available to reports/heatmaps without re-joining the source. (Resolves the old "metadata file scope" open question.)

## Settled decisions

- Six stages, run individually, connected by Snakemake (Stage 1 ingestion is outside Snakemake — user-written bridge).
- Folds are **NOT** generated in preprocessing — they belong to Model Training, to support the seed sweep (independent model-seed × fold-seed variation).
- **No fitted statistics in bundles** — bundles carry raw labels/embeddings; normalization etc. is computed at training time from the training fold only. Combined with holdout patients filtered out of all folds → leakage-free.
- Embedding reuse via **content-addressed cache** (key: coords + patch size + resolution + embedding model + source variant + augmentation).
- Pipeline reads a **manifest** (IDs → paths), never the on-disk layout — so folder vs flat ingestion both work.

## Storage formats

Binary is the source of truth; GeoJSON is the TissUUmaps view (pipeline never computes from GeoJSON).
- Embeddings → HDF5. Patch coordinates → HDF5. Tissue outlines → **polygon arrays** (+ GeoJSON export). Transform matrices → JSON.
- **BEAM** = the project's own per-biopsy, per-model evaluation result format. Stored as HDF5, `{biopsy_id}__{model_id}.beam.h5`. Contains attention (raw/sigmoid/rank), prediction, labels, patch coords, outline (polygon array, divisible into quartiles), metadata. Appendable by design. (Name is provisional — easy to rename.)
- Detailed format specs live in `docs/formats/` (beam, embeddings-and-patches, outlines) — kept separate so design docs stay overview-level. Stage docs give rough bullets + a link.

## Reports & experiments

- Reports = regenerable **view** over manifests/`runs.parquet`/BEAM; static HTML, **Plotly** interactive plots, standalone `report_assets/reports.css` (no MkDocs dep), hybrid layout (self-contained run folders + index files).
- **Runs are generated, not configured.** `model_experiment.yaml` = shared `defaults` + explicit `runs` (each a run_id overriding e.g. bundle or hyperparameters), each fanning out over the seed sweep. **HPO is a SEPARATE config** (`hpo.yaml`) that fans out into trials. Config count = O(experiments), not O(models). Each run → a run record → aggregated into `results/runs.parquet` (master tagged, exportable table). Reports index by **tags** (faceted: model experiment/bundle/architecture/stain), never by name-parsing.
- **HPO is segregated**: under `results/experiments/{exp}/hpo/` with its own index; `keep_checkpoints: all|top_n|none` (default top_n). Seed-sweep models live under `sweep/` (the keepers). Workflow: HPO explores → promote best N → seed sweep on them.
- **Table export, two levels**: client-side CSV/JSON buttons + link to canonical backing artifact (CSV/Parquet) at a stable path for the user's own Python scripts.
- High-value pre-model reports (independent of training, build first): fold×cohort composition, dataset distribution, cohort composition.
- Design: `docs/design/11-reports.md`; config: `docs/configs/reports.yaml`.

## Configuration

- **`base.yaml`** holds shared roots (raw/normalized/processed/bundles/results) + registry paths + rarely-edited settings; layered UNDER every stage config (base first, stage overrides win). Changing an output path = one edit.
Draft configs in `docs/configs/` (base, cohorts, seeds, wsi_transformation, preprocessing, model_experiment, hpo, evaluation, heatmaps, reports). All marked DRAFT. Each has a wrapper md page that embeds the YAML via pymdownx.snippets; listed under a collapsible Configuration nav menu. Overview in `docs/design/10-configuration.md`. Per-file (not monolithic) because stages run individually; single merged config left as an open alternative.
- **`seeds.yaml` is a project-wide named registry**: top-level `seed_sets` maps names → split configs (each references a `cohort`); a run picks one via `seed_set:`. Models sharing a name get identical splits.
- **Label balancing** lives in `model_experiment.yaml` (per fold, train split only).
- **Augmentation lives in PREPROCESSING, not training** — histology augmentation (flips/rotations/stain jitter) changes pixels, so it runs the embedding foundation model on augmented patches and caches each variant as its own embedding set (augmentation_id is part of the cache key). Training only picks whether to sample them (`use_augmented_embeddings`).

## Six stages

1. Data Ingestion (outside Snakemake) — normalize raw → standard format + per-biopsy label CSV
2. WSI Transformation — registration (raw/rigid/elastic) + tissue outlines + cross-stain intersection
3. Dataset Preprocessing — label processing, patch generation, patch embedding (+cache), bundle prep (= prepared cohort)
4. Model Training — fold generation (split registry `seeds.yaml`), model experiment (defaults + runs), seed sweep; HPO separate + segregated
5. Evaluation — inference, per-biopsy aggregation into BEAM files, reports
6. Heatmap Generation — attention overlays (PNG) + TissUUmaps GeoJSON. NOTE: attention is in the raw frame, so coords must be pushed through the variant's transform matrix to overlay on rigid/elastic.

## Doc structure

Design: `01-overview` · `02-data-model` · `03-data-ingestion` · `04-wsi-transformation` · `05-dataset-preprocessing` · `06-model-training` · `07-evaluation` · `08-heatmaps` · `09-open-questions` · `10-configuration` · `11-reports` · `12-appendix` (decisions + old-pipeline baseline)
Formats: `docs/formats/{beam,embeddings-and-patches,outlines}.md`
Configs: `docs/configs/*.yaml` (drafts): base, cohorts, seeds, wsi_transformation, preprocessing, model_experiment, hpo, evaluation, heatmaps, reports
