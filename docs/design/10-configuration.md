# Configuration

The Snakemake-driven stages each take a YAML config. Since stages run individually (and increasingly often), each has its own config file rather than one monolith. Stage 1 (Data Ingestion) is outside Snakemake and user-written, so it has no config here.

The full files are shown on their own pages (under **Configuration** in the sidebar) and embedded directly from `docs/configs/` — the YAML you see *is* the file.

!!! warning "These are drafts"
    Field names and structure will change as stages are implemented.

## Config files

| Config | Stage | Controls |
|---|---|---|
| [`base.yaml`](../configs/base.md) | all | Shared **roots** + rarely-edited settings; layered under every stage config |
| [`cohorts.yaml`](../configs/cohorts.md) | 3 | Named cohorts (multi-dataset) + per-patient **roles**; resolved/validated/reported by `resolve_cohort` |
| [Split registry (`seeds.yaml`)](../configs/seeds.md) | 4 | Named seed/split sets referencing a cohort; shared by all models |
| [`wsi_transformation.yaml`](../configs/wsi_transformation.md) | 2 | Variants to produce, registration backend, outline + quartile options |
| [`preprocessing.yaml`](../configs/preprocessing.md) | 3 | Derived labels, patching, embedding, **augmentation**, **bundle = prepared cohort** |
| [`model_experiment.yaml`](../configs/model_experiment.md) | 4 | Shared defaults + **runs** (run_id overrides), `subset`, `seed_set`, **balancing**, sweep |
| [`hpo.yaml`](../configs/hpo.md) | 4 | Separate hyperparameter search; segregated outputs, top-N kept |
| [`evaluation.yaml`](../configs/evaluation.md) | 5 | Bundle + model, **subset** (holdout), attention variants, output |
| [`heatmaps.yaml`](../configs/heatmaps.md) | 6 | Source variant, attention type, colormap, render + output options |
| [`reports.yaml`](../configs/reports.md) | — | Plot backend, standalone CSS, **HPO segregation**, table export |

## Cross-cutting notes

- **`base.yaml` holds the roots.** Output paths, data roots, and registry locations live there once; stage configs are layered on top (base first, stage overrides win) and reference the roots instead of hard-coding paths.
- **Cohorts are the root.** `cohorts.yaml` defines named, possibly multi-dataset patient groups; within each, patients have a **role** (`development` / `holdout`). Both splits and bundles derive from a cohort.
- **Split registry references a cohort, not a dataset.** `seeds.yaml` holds named seed/split configs under `seed_sets`; each names a `cohort`, and folds are computed over its development patients. A change in the cohort's frozen membership raises a stale-splits warning.
- **A bundle is a prepared cohort.** `preprocessing.yaml` assembles one bundle per cohort × stain × embedding × variant; every patient is present, each bag tagged with its `role`. Stages pick a **subset** (`development` / `holdout` / `all`) — the union is `all`, never a hand-built list.
- **`model_experiment.yaml` is shared defaults + explicit runs**, each fanning out over the seed sweep — config count stays O(experiments), not O(models). HPO is a **separate** config with segregated outputs. See [Reports](11-reports.md).
- **Augmentation lives in preprocessing**, not training — it runs the embedding foundation model on augmented patches and caches each variant as its own embedding set. Training only chooses whether to sample those sets.
- **Balancing** lives in `model_experiment.yaml` and applies per fold from the training split only.
- A single merged `config.yaml` with per-stage sections is a possible alternative if the per-file split proves cumbersome — left open.
