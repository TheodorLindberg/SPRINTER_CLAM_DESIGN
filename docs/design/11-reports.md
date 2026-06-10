# Reports

Interactive HTML reports over the pipeline's artifacts. Configured by [`reports.yaml`](../configs/reports.md).

## Principle: a report is a view, never a source of truth

Every report is regenerated from artifacts the pipeline already produces — manifests, fold assignments, metrics, [BEAM](../formats/beam.md) files, and the **runs index**. Reports store nothing new, so they are disposable and reproducible: delete the report folder and rebuild anytime. Each page shows the provenance it was built from (config, git commit, membership hash, seeds).

## Format

- **Static HTML** — no server; versionable, zippable, archival.
- **Interactive plots via Plotly** wherever possible — embedded self-contained, still interactive offline.
- **Sortable/filterable tables** (client JS) for run and fold tables.
- **One standalone `reports.css`** — no MkDocs/Material dependency.
- **WSI / heatmaps** link out to TissUUmaps; a thumbnail PNG is embedded inline.

---

## The model-experiment umbrella and the runs index

Runs come from a [model experiment](../configs/model_experiment.md) (shared defaults + explicit runs, each fanning out over the seed sweep) and from [HPO](../configs/hpo.md) (a search that fans out into trials) — so config count stays O(experiments), never O(models).

```
Model experiment   (named umbrella, e.g. ki67_stain_comparison — may span many bundles)
  └─ Run      (= bundle × architecture × target × fold_seed × model_seed [× hpo_trial])
```

Each run emits a small **run record** (`run.json`: its tags + metrics + artifact paths). All records aggregate into **`runs.parquet`** — the master, tagged table that the whole report reads from. Indexing is driven by these tags, **not by parsing names**: the report builds **faceted indexes** (group-by experiment, bundle, architecture, stain, …) as different views of the same records. `runs.parquet` is also directly exportable — it is the table you'd load to do cross-experiment analysis.

---

## Results folder layout

```text
results/                          # roots.results (base.yaml)
  runs.parquet                    # master runs index — every run, tagged (export backbone)
  reports/
    index.html                    # top-level, cross-experiment
    datasets/{dataset_id}.html    # distributions
    cohorts/{cohort}.html         # role composition (development / holdout)
    splits/{seed_set}.html        # fold × cohort composition
  experiments/
    {experiment}/
      index.html                  # faceted: by bundle / architecture / stain
      sweep/                      # seed-sweep models — the keepers, easy to find
        {run_id}/
          checkpoints/  metrics/  report/index.html
      hpo/                        # SEGREGATED — rarely inspected
        index.html                # Optuna plots + exportable trials table
        trials/{trial_id}/        # kept per reports.yaml keep_checkpoints (all|top_n|none)
```

### HPO is kept apart from the seed sweep

HPO can produce hundreds of models you'll rarely revisit, while the **seed-sweep** models are the ones you actually keep and inspect. So:

- HPO models and report live under `experiments/{exp}/hpo/`, with their **own index** — separate from `sweep/`.
- `reports.yaml → hpo.keep_checkpoints` controls storage (`all` / `top_n` / `none`); by default only the **top-N** HPO checkpoints are retained.
- Workflow: HPO explores → promote the best N hyperparameters → run a **seed sweep** on them. The sweep output is the durable result; HPO is exploratory scaffolding.
- The HPO index is a **summary**, not hundreds of pages: optimization-history, parallel-coordinates, and param-importance plots plus a sortable trials table; full detail pages only for the promoted top-N.

---

## High-value pages

- **Fold × cohort composition** (`splits/{seed_set}.html`) — per fold_seed, every patient with fold number, train/val/test role, and cohort. Makes leakage **visually auditable**: holdout sits outside every fold, no patient spans train/val/test. Needs no model results — cheap to build right after split generation.
- **Dataset distribution** — label histograms, counts per dataset/stain/cohort, missingness.
- **Cohort composition** — development vs holdout membership.

These pre-model reports are independent of all training, so they are the natural **first thing to build** — early validation that cohorts and splits behave before any model runs.

---

## Data export

Reports are a window onto exportable artifacts, at two levels:

1. **Client-side export** — every table has CSV / JSON export buttons for what's shown.
2. **Backing-artifact link** — every table also links the **canonical CSV/Parquet** it was rendered from (full precision, all columns), at a **stable, documented path** so a script can find it programmatically (`runs.parquet`, fold-composition CSV, predictions parquet, …).

So when a table is useful for an article, you grab the real backing data and run your own Python on it — not a scraped HTML table.

---

## Generation

Reports emit at two moments, sharing the same components (breadcrumbs, provenance block, table macro, plot-card macro):

- **Pre-model** — dataset distribution, **cohort composition (emitted by `resolve_cohort`)**, fold composition (after split generation).
- **Post-model** — seed-sweep metrics, HPO summary, per-biopsy evaluation pages (after training / evaluation).

A final aggregation step stitches the top-level cross-experiment index.
