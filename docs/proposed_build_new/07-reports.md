# 07 · Reports toolkit

The `histomil.reporting` stage builds the report layer from
[`docs/design/11-reports.md`](../design/11-reports.md): static HTML, Plotly plots, standalone
CSS, faceted indexes over `runs.parquet`, two-level export. It reads BEAM through
`shared.formats.beam.BeamReader` — so it never imports `histomil.evaluation`. A report is a
**view, never a source of truth** — regenerable from manifests, `runs.parquet`, and BEAM, storing
nothing new.

## Toolkit in `shared`, page builders in the stages

`resolve_cohort` (in **preprocessing**) emits the cohort HTML report so a cohort can be eyeballed
*before* heavy compute. So the **toolkit primitives** live in `histomil.shared.report` (behind the
`histomil[reports]` extra: plotly + jinja2), and any stage may emit a page with them without
importing `histomil.reporting`:

- **Toolkit primitives** (`jinja2` env, plot-card + table macros, provenance block, `reports.css`)
  → `histomil.shared.report`.
- **The cohort page builder** → `histomil.preprocessing.cohort_report` (runs at cohort-resolution
  time), using the shared toolkit.
- **`histomil.reporting`** → the cross-cutting and post-model pages (faceted experiment indexes,
  HPO summary, per-biopsy evaluation, the cross-experiment stitch).

So the toolkit is shared; *which* pages exist and *what* they plot is owned by the stage that emits
them. Below, "module" means a builder in `histomil.reporting` unless noted.

## Toolkit (`histomil.shared.report`)

A small shared layer every page reuses, so pages stay declarative:

```python
class Report:
    env: jinja2.Environment            # templates + macros
    def plot_card(self, fig: plotly.graph_objects.Figure, title) -> str   # embedded, self-contained
    def table(self, df, *, export=("csv","json"), backing: Path|None) -> str
    def provenance_block(self, prov: Provenance, *, membership_hash, seeds) -> str
    def breadcrumbs(self, *crumbs) -> str
    def write(self, html: str, path: Path) -> None
```

- **Plotly** figures embedded self-contained (interactive offline), one `<script>` include.
- **Tables** render with client-side sort/filter + CSV/JSON export buttons **and** a link to the
  canonical backing CSV/Parquet (full precision) at a stable path — the two-level export the docs
  require, so an article grabs the real artifact, not a scraped table.
- **`histomil/shared/report/assets/reports.css`** is the one standalone stylesheet (no
  MkDocs/Material dependency); mirrors the repo's existing `report_assets/`.
- **Provenance block** on every page: config hash, git commit, membership hash, seeds — from
  `shared.provenance`.

## Pages → modules

```text
results/
  runs.parquet                       # master index — train/aggregate.py writes it
  reports/
    index.html                       # reports/index.py — cross-experiment top level
    datasets/{dataset_id}.html       # reports/datasets.py
    cohorts/{cohort}.html            # reports/cohorts.py  (emitted by resolve_cohort)
    splits/{seed_set}.html           # reports/splits.py   (after generate_folds)
  experiments/{experiment}/
    index.html                       # reports/experiments.py — FACETED by tags
    sweep/{run_id}/report/index.html
    hpo/index.html                   # reports/hpo.py — SEGREGATED summary + trials table
```

| Module | Page | Built when | Reads |
|---|---|---|---|
| `reporting.datasets` | label histograms, counts, missingness | pre-model | manifests + derived labels |
| `preprocessing` (cohort builder) | members by dataset × role, label dist per role, membership_hash | by `resolve_cohort` | `membership.csv` |
| `splits.py` | per fold_seed: every patient × fold × split × cohort | after `generate_folds` | folds CSV + membership |
| `experiments.py` | faceted seed-sweep metrics (by bundle/architecture/stain) | post-model | `runs.parquet` |
| `hpo.py` | optimization-history, parallel-coords, param-importance + trials table; detail only for top-N | post-HPO | Optuna study + `runs.parquet` |
| `evaluation.py` | per-biopsy pages: prediction vs label, attention thumbnail → TissUUmaps | post-eval | BEAM files |
| `index.py` | cross-experiment stitch | final | all of the above |

## Faceting, not name-parsing

The report **never parses run names**. `experiments.py` reads `runs.parquet` and groups by tag
columns (`model_experiment`, `bundle_id`, `architecture.family`, `stain`, …) to build each
faceted view — the same records, many lenses. This is why the run record
([06](06-formats-and-schemas.md#json-artifacts)) flattens all tags into columns.

## HPO segregation

HPO can emit hundreds of rarely-revisited models; the seed sweep holds the keepers. So the HPO
index lives under `experiments/{exp}/hpo/` separate from `sweep/`, is a **summary** (plots +
sortable trials table, full pages only for the promoted top-N), and retention follows
`reports.yaml → hpo.keep_checkpoints` (`all`/`top_n`/`none`, default top-N).

## Build order (high value first)

The **pre-model** pages (dataset distribution, cohort composition, fold composition) are
independent of all training and make leakage visually auditable — holdout outside every fold, no
patient spanning train/val/test. They are the natural first thing to build and the earliest
validation that cohorts/splits behave, so they lead the [roadmap](08-testing-and-roadmap.md).

## Generation

Two moments, sharing the toolkit macros:

- **Pre-model** — `report build --section dataset_distribution|cohort_composition|fold_composition`
  (cohort page also auto-emitted by `resolve_cohort`).
- **Post-model** — seed-sweep metrics, HPO summary, per-biopsy evaluation pages, then `index.py`
  stitches the cross-experiment top level.

Driven by `reports.yaml` (`plots`, `css`, `layout: hybrid`, `include:` section toggles, `tables`
export, `hpo` segregation).
