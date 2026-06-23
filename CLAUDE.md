# Project context

Histology training & evaluation pipeline for MIL models on whole-slide images.

Design docs in `docs/design/` are the **authoritative source of truth**.
Always read the relevant design doc before suggesting implementation changes.

## Three documentation layers (progressive depth)

The goal is a spec deep enough to **rebuild the pipeline from scratch** against it.
- **Overview** (`docs/design/`) — concept, In→Out, decisions, diagrams. Keep shallow; don't bloat. Each stage page has a one-line *Go deeper* pointer.
- **Specification** (`docs/spec/`) — exact, implementation-independent contracts: schemas (cols/dtypes/layouts), config field semantics, **invariants**, **acceptance criteria**. The testable WHAT.
- **Implementation** (`docs/impl/`) — algorithms (incl. math), signatures, library choices, pseudocode. The HOW; changeable as long as the spec holds.
Built out so far for: WSI transformation (incl. the **PCA biopsy-axis** line), preprocessing, training. Extend stage by stage.

Old pipeline code for reference (gitignored, never referenced by the published docs): `clam_old_pipeline/` — registration code in `clam_old_pipeline/registration_and_preprocessing/` (VALIS). Review/migration notes in `OLD_PIPELINE_REVIEW.md` (repo root, outside docs).

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
- Configs: `cohorts.yaml` (members + holdout designation → frozen hashed membership), `seeds.yaml` references `cohort:`, `pipeline.yaml`'s `preprocessing.bundle` = prepared cohort, the experiment config carries `subset:` (the evaluation subset is a CLI target). No `patient_exclusion` (lifted to the cohort's holdout role).
- **One stain per bundle for now** (one model per stain → evaluation per-bag→per-biopsy is 1:1). Multi-stain/multi-target (e.g. one model on Ki67+PSA embeddings predicting both scores) is a documented future extension — see `09-open-questions.md#stains-per-bundle`.
- **Cohort is an actual pipeline step**: `resolve_cohort` rule (first in preprocessing, `cohort` target) freezes the hashed membership, validates (members exist, holdout clean, pooled labels comparable), and emits `results/reports/cohorts/{cohort}.html`. Runnable before heavy compute. Gates `assemble_bundle` + `generate_folds`.

## Metadata flow (decided)

The scan manifest (Stage 1 ingestion, IDs → WSI path) is the pipeline contract. Metadata rides on it as columns **at any entity level** (patient/biopsy/scan — coarser levels just repeat across rows); no separate metadata file. Preprocessing **forwards** this metadata into the bundle manifest and into each BEAM `/metadata`, so it's available to reports/heatmaps without re-joining the source. (Resolves the old "metadata file scope" open question.)

## Settled decisions

- Six stages, run individually, connected by Snakemake (Stage 1 ingestion is outside Snakemake — user-written bridge).
- Folds are **NOT** generated in preprocessing — they belong to Model Training, to support the seed sweep (independent model-seed × fold-seed variation).
- **No fitted statistics in bundles** — bundles carry raw labels/embeddings; normalization etc. is computed at training time from the training fold only. Combined with holdout patients filtered out of all folds → leakage-free.
- Embedding reuse via a **file-level cache**: one HDF5 per (scan · source variant · embedding model · patch config); the file path is the key, so Snakemake tracks reuse directly (no separate store). Reused across cohorts since the cohort isn't in the path.
- Pipeline reads a **manifest** (IDs → paths), never the on-disk layout — so folder vs flat ingestion both work.

## Storage formats

Binary is the source of truth; GeoJSON is the TissUUmaps view (pipeline never computes from GeoJSON).
- Embeddings → HDF5. Patch coordinates → HDF5. Tissue outlines → **polygon arrays** (+ GeoJSON export). Transform matrices → JSON.
- **BEAM** = the project's own per-biopsy, per-**sweep** evaluation result format. Stored as HDF5, `{biopsy_id}__{run_family}.beam.h5` — one BEAM covers every model (fold_seed × model_seed) of one seed sweep, each contributing its own prediction + attention (raw/sigmoid/rank) + full stats under `models/{run_id}/`. Patch coords, outline (polygon array, divisible into quartiles), labels, and metadata are shared by every model in the file. Appendable by design. (Name is provisional — easy to rename.)
- Detailed format specs live in `docs/formats/` (beam, embeddings-and-patches, outlines) — kept separate so design docs stay overview-level. Stage docs give rough bullets + a link.

## Reports & experiments

- Reports = regenerable **view** over manifests/`runs.parquet`/BEAM; static HTML, **Plotly** interactive plots, standalone `report_assets/reports.css` (no MkDocs dep), hybrid layout (self-contained run folders + index files).
- **Runs are generated, not configured.** The experiment config (one file per experiment) = shared `defaults` + explicit `runs` (each a run_id overriding e.g. bundle or hyperparameters), each fanning out over the seed sweep. **HPO is an optional `hpo` block** in the same experiment file that fans out into trials. Config count = O(experiments), not O(models). Each run → a run record → aggregated into `results/runs.parquet` (master tagged, exportable table). Reports index by **tags** (faceted: model experiment/bundle/architecture/stain), never by name-parsing.
- **HPO is segregated**: under `results/experiments/{exp}/hpo/` with its own index; `keep_checkpoints: all|top_n|none` (default top_n). Seed-sweep models live under `sweep/` (the keepers). Workflow: HPO explores → promote best N → seed sweep on them.
- **Table export, two levels**: client-side CSV/JSON buttons + link to canonical backing artifact (CSV/Parquet) at a stable path for the user's own Python scripts.
- High-value pre-model reports (independent of training, build first): fold×cohort composition, dataset distribution, cohort composition.
- Design: `docs/design/11-reports.md`; config: the `reports` section of `pipeline.yaml`.

## Configuration

- **`base.yaml`** holds shared roots (raw/normalized/processed/bundles/results) + registry paths + rarely-edited settings; layered UNDER every stage config (base first, stage overrides win). Changing an output path = one edit.
Draft configs in `docs/configs/` (base, cohorts, seeds, pipeline, experiment). All marked DRAFT. Each has a wrapper md page that embeds the YAML via pymdownx.snippets; listed under a collapsible Configuration nav menu. Overview in `docs/design/10-configuration.md`. Organized by edit cadence: infra (`base.yaml`), append-only registries (`cohorts.yaml`/`seeds.yaml`), stage defaults (`pipeline.yaml` = `wsi_transformation` + `preprocessing` + `reports` sections), and one file per experiment (`experiments/<name>.yaml` = defaults + runs + optional `hpo` block). Selection (which dataset/model/subset/BEAM) is a CLI target, not config.
- **`seeds.yaml` is a project-wide named registry**: top-level `seed_sets` maps names → split configs (each references a `cohort`); a run picks one via `seed_set:`. Models sharing a name get identical splits.
- **Label balancing** lives in the experiment config (per fold, train split only).
- **No augmentation in v1.** Foundation-model embeddings are largely invariant to histology transforms, so augmentation isn't built; if added later it lives in preprocessing (runs the foundation model on augmented patches). See `docs/design/09-open-questions.md#augmentation`.

## Six stages

1. Data Ingestion (outside Snakemake) — normalize raw → standard format + per-biopsy label CSV
2. WSI Transformation — registration (raw/rigid/elastic) + tissue outlines + cross-stain intersection
3. Dataset Preprocessing — **cohort resolution** (resolve_cohort: freeze+validate+report), label processing, patch generation, patch embedding (+ file-level cache), bundle prep (= prepared cohort)
4. Model Training — fold generation (split registry `seeds.yaml`), experiment config (defaults + runs), seed sweep; HPO optional block, segregated outputs
5. Evaluation — inference, per-biopsy aggregation into BEAM files, reports
6. Heatmap Generation — attention overlays (PNG) + TissUUmaps GeoJSON. NOTE: attention is in the raw frame, so coords must be pushed through the variant's transform matrix to overlay on rigid/elastic.

## Environment & tooling

`docs/environment.md`. Container = Apptainer/Singularity `.sif`. **Production: self-contained SIF** (all Python deps baked in, immutable/reproducible). **Dev: SIF + mounted `uv` venv** (decided approach — fast add/test of packages without rebuild; bake into SIF to promote). Env vars: `HF_TOKEN` (gated models), `HF_HUB_OFFLINE=1`, SSL workarounds (`HF_HUB_DISABLE_SSL_VERIFICATION=1`, unset `SSL_CERT_FILE`), `CUDA_VISIBLE_DEVICES`. SLURM: launch script submits the Snakemake controller → per-rule worker jobs in the container. TissUUmaps viewer via docker (`tissuumaps/tissuumaps`, port 5100).

## Doc structure

Design: `01-overview` · `02-data-model` · `glossary` (all terms + abbreviations) · `03-data-ingestion` · `04-wsi-transformation` · `05-dataset-preprocessing` · `06-model-training` · `07-evaluation` · `08-heatmaps` · `09-open-questions` · `10-configuration` · `11-reports` · `12-appendix` (design decisions / rationale)
Formats: `docs/formats/{beam,embeddings-and-patches,outlines}.md`
Configs: `docs/configs/*.yaml` (drafts): base, cohorts, seeds, pipeline (wsi_transformation + preprocessing + reports), experiment (defaults + runs + optional hpo)
