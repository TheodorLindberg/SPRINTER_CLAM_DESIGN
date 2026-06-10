# SPRINTER · CLAM Design Docs

Design documentation for a **histology MIL training & evaluation pipeline** — preparing whole-slide-image datasets, generating patch embeddings, training and evaluating models, and producing reports and heatmaps.

📖 **Read the docs:** <https://theodorlindberg.github.io/SPRINTER_CLAM_DESIGN/>

> Status: working draft. The docs are the authoritative source of truth for the design; exact schemas are deliberately deferred until implementation.

## What's inside

- **Overview & Data Model** — the six-stage pipeline, core entities, identifiers, and a [glossary](https://theodorlindberg.github.io/SPRINTER_CLAM_DESIGN/design/glossary/) of every term and abbreviation.
- **Pipeline stages** — one page each: Data Ingestion → WSI Transformation → Dataset Preprocessing → Model Training → Evaluation → Heatmap Generation. Every stage opens with an *In → Out* summary.
- **Formats** — file-format specs (BEAM evaluation results, embeddings & patches, outlines).
- **Configuration** — draft per-stage Snakemake config files, each embedded and explained.
- **Reports**, **Open Questions**, and an **Appendix** of design decisions.

## Read it

- **Online** — the hosted site above is the easiest way to read and search.
- **Locally** — build it yourself (below) to preview edits before pushing.

## Build locally

Requires Python 3. Install the one dependency and serve with live reload:

```bash
pip install mkdocs-material
mkdocs serve          # http://127.0.0.1:8000 — reloads on save
```

Build the static site into `site/`:

```bash
mkdocs build
```

## Deployment

The site is **published automatically** to GitHub Pages on every push to `main`, via [`.github/workflows/deploy-docs.yml`](.github/workflows/deploy-docs.yml). No manual deploy step is needed.

## Repository layout

```text
docs/
  index.md
  design/      # overview, data model, glossary, the 6 stages, reports, appendix
  configs/     # draft Snakemake configs (YAML) + explained wrapper pages
  formats/     # BEAM, embeddings & patches, outlines
report_assets/ # standalone CSS for generated reports
mkdocs.yml     # site config (Material theme, Mermaid, snippets)
CLAUDE.md      # working context for the design
```
