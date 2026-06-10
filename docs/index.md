# Histology Training & Evaluation Pipeline

Design specification for a pipeline that prepares histology datasets, generates patch embeddings, trains MIL models, evaluates them, and produces reports and heatmaps.

Status: **working draft.**

## Reading order

| Document | Contents |
|---|---|
| [Overview](design/01-overview.md) | Purpose, scope, the five stages, training-vs-heatmap principle |
| [Data Model](design/02-data-model.md) | Entities, identifiers, bag naming, labels, source variants |
| [Stage 1 · Data Ingestion](design/03-data-ingestion.md) | Normalizing raw datasets |
| [Stage 2 · WSI Transformation](design/04-wsi-transformation.md) | Registration variants and outlines |
| [Stage 3 · Dataset Preprocessing](design/05-dataset-preprocessing.md) | Labels, patches, embeddings, bundles |
| [Stage 4 · Model Training](design/06-model-training.md) | Folds, seed sweep, HPO, training function |
| [Stage 5 · Evaluation](design/07-evaluation.md) | Inference, aggregation, BEAM result format |
| [Stage 6 · Heatmap Generation](design/08-heatmaps.md) | Attention overlays + TissUUmaps export |
| [Configuration](design/10-configuration.md) | Draft Snakemake config files per stage |
| [Open Questions & Decisions](design/09-open-questions.md) | Resolved defaults and undecided items |

Detailed file-format specs (BEAM, embeddings, outlines) live under [formats/](formats/beam.md).
