# Configuration

The Snakemake-driven stages each take a YAML config. Since stages run individually (and increasingly often), each has its own config file rather than one monolith. Stage 1 (Data Ingestion) is outside Snakemake and user-written, so it has no config here.

The full files are shown on their own pages (under **Configuration** in the sidebar) and embedded directly from `docs/configs/` — the YAML you see *is* the file.

!!! warning "These are drafts"
    Field names and structure will change as stages are implemented.

## Config files

| Config | Stage | Controls |
|---|---|---|
| [`base.yaml`](../configs/base.md) | all | Shared **roots** + rarely-edited settings; layered under every stage config |
| [`patient_sets.yaml`](../configs/patient_sets.md) | — | Named patient sets (multi-dataset) + **cohort** partition (development / holdout) |
| [Split registry (`seeds.yaml`)](../configs/seeds.md) | 4 | Named seed/split sets referencing a patient set; shared by all models |
| [`wsi_transformation.yaml`](../configs/wsi_transformation.md) | 2 | Variants to produce, registration backend, outline + quartile options |
| [`preprocessing.yaml`](../configs/preprocessing.md) | 3 | Derived labels, patching, embedding, **augmentation**, **bundle assembly** from a patient set |
| [`training.yaml`](../configs/training.md) | 4 | Bundle, `seed_set`, **cohort scope**, target, architecture, **balancing**, sweep, HPO |
| [`evaluation.yaml`](../configs/evaluation.md) | 5 | Bundle + model, **cohort scope** (holdout), attention variants, output |
| [`heatmaps.yaml`](../configs/heatmaps.md) | 6 | Source variant, attention type, colormap, render + output options |
| [`reports.yaml`](../configs/reports.md) | — | Plot backend, standalone CSS, **HPO segregation**, table export |

## Cross-cutting notes

- **`base.yaml` holds the roots.** Output paths, data roots, and registry locations live there once; stage configs are layered on top (base first, stage overrides win) and reference the roots instead of hard-coding paths.
- **`training.yaml` is an experiment definition** that fans out into many runs (seed sweep × HPO) — config count stays O(experiments), not O(models). See [Reports](11-reports.md).
- **Patient sets are the root.** `patient_sets.yaml` defines named, possibly multi-dataset patient collections, each partitioned into a `development` and a `holdout` cohort. Both splits and bundles derive from a patient set.
- **Split registry references a patient set, not a dataset.** `seeds.yaml` holds named seed/split configs under `seed_sets`; each names a `patient_set`, and folds are computed over its development cohort. A change in the set's frozen membership raises a stale-splits warning.
- **Bundles carry cohorts as tags, not as separate bundles.** `preprocessing.yaml` assembles one bundle per patient set × stain × embedding × variant; each bag is tagged `development` / `holdout`. Stages pick a **cohort scope** (`development` / `holdout` / `all`) — the union is `all`, never a hand-built list.
- **Augmentation lives in preprocessing**, not training — it runs the embedding foundation model on augmented patches and caches each variant as its own embedding set. Training only chooses whether to sample those sets.
- **Balancing** lives in `training.yaml` and applies per fold from the training split only.
- A single merged `config.yaml` with per-stage sections is a possible alternative if the per-file split proves cumbersome — left open.
