# Configuration

The Snakemake-driven stages take YAML configs, **layered**: `base.yaml` is always loaded first, then the file for what you're running (`--configfile base.yaml pipeline.yaml`), with later values winning. Stage 1 (Data Ingestion) is outside Snakemake and user-written, so it has no config here.

The full files are shown on their own pages (under **Configuration** in the sidebar) and embedded directly from `docs/configs/` — the YAML you see *is* the file.

!!! warning "These are drafts"
    Field names and structure will change as stages are implemented.

## Config files

Organized by how often they are edited, not by stage:

| Config | Tier | Controls |
|---|---|---|
| [`base.yaml`](../configs/base.md) | infrastructure (set once) | Shared **roots**, registry locations, `defaults` (`source_variant`, `reference_stain`), and set-once `evaluation` / `heatmaps` defaults |
| [`cohorts.yaml`](../configs/cohorts.md) | registry (append-only) | Named cohorts (multi-dataset) + per-patient **roles**; resolved/validated/reported by `resolve_cohort` |
| [Split registry (`seeds.yaml`)](../configs/seeds.md) | registry (append-only) | Named seed/split sets referencing a cohort; shared by all models |
| [`pipeline.yaml`](../configs/pipeline.md) | stage defaults (occasional) | `wsi_transformation`, `preprocessing` (labels, patching, embedding, **bundle = prepared cohort**), and `reports` — as independent sections |
| [`experiments/<name>.yaml`](../configs/experiment.md) | per experiment (frequent) | One file per experiment: shared `defaults` + explicit **runs** (`run_id` overrides, `subset`, `seed_set`, **balancing**), plus an optional **`hpo`** search block |

**Selection vs. definition.** Configs hold *definitions and defaults*; *what you're running now* — which dataset to transform, which model/subset to evaluate, which BEAM to render — is a **command-line target**, not a config field. So there are no standalone evaluation/heatmap config files: their defaults live in `base.yaml`, and the selection is a CLI argument.

## Cross-cutting notes

- **`base.yaml` holds the roots and set-once defaults.** Data roots, output paths, registry locations, and evaluation/heatmap defaults live there once; everything else layers on top and refers to the roots instead of hard-coding paths — so changing an output location is a single edit.
- **The cohort is the root entity.** `cohorts.yaml` defines named, possibly multi-dataset patient groups; within each, every patient has a role (`development` or `holdout`). Both splits and bundles derive from a cohort. Cohorts are **append-only** — version with a new name (`..._v2`) rather than editing a frozen entry, since membership is hashed and an edit stales every split built from it.
- **The split registry references a cohort, not a dataset.** `seeds.yaml` holds named seed/split sets; each names a cohort, and folds are computed over its development patients. A membership-hash change raises a stale-splits warning.
- **A bundle is a prepared cohort.** `pipeline.yaml`'s `preprocessing.bundle` assembles one bundle per cohort × stain × embedding × variant, every patient present and tagged by role. Stages pick a subset (`development`, `holdout`, or `all`) — the union is `all`, never a hand-built list.
- **An experiment is shared defaults plus explicit runs**, each fanning out over the seed sweep, so config count stays O(experiments), not O(models). A run references its `bundle` by components (cohort · stain · variant · embedding); the bundle id is derived, not hand-typed. HPO is an optional block in the same file.

A run's resolved config (base + pipeline/experiment + any CLI overrides) is recorded with its results, so a result stays reproducible regardless of later edits to the source files.
