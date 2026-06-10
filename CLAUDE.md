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

## Configuration

Draft per-stage Snakemake configs in `docs/configs/` (wsi_transformation, preprocessing, training, evaluation, heatmaps) + `seeds.yaml` split registry. All marked DRAFT. Overview in `docs/design/10-configuration.md`. Per-file (not monolithic) because stages run individually; single merged config left as an open alternative.
- Label balancing + augmentation live in `training.yaml` / Stage 4 doc. Augmentation defaults to bag/embedding level because embeddings are cached — image-space augmentation would require re-embedding under new cache keys.

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
