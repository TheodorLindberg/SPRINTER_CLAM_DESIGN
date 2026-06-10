# Histology Training & Evaluation Pipeline

Design specification for a pipeline that prepares histology datasets, generates patch embeddings, trains MIL models, evaluates them, and produces reports and heatmaps.

Status: **working draft.**

## Reading order

| Document | Contents |
|---|---|
| [Overview](design/01-overview.md) | Purpose, scope, the five stages, training-vs-heatmap principle |
| [Data Model](design/02-data-model.md) | Entities, identifiers, bag naming, labels, source variants |
| [Glossary](design/glossary.md) | All terms + abbreviations in one place |
| [Stage 1 · Data Ingestion](design/03-data-ingestion.md) | Normalizing raw datasets |
| [Stage 2 · WSI Transformation](design/04-wsi-transformation.md) | Registration variants and outlines |
| [Stage 3 · Dataset Preprocessing](design/05-dataset-preprocessing.md) | Labels, patches, embeddings, bundles |
| [Stage 4 · Model Training](design/06-model-training.md) | Folds, seed sweep, HPO, training function |
| [Stage 5 · Evaluation](design/07-evaluation.md) | Inference, aggregation, BEAM result format |
| [Stage 6 · Heatmap Generation](design/08-heatmaps.md) | Attention overlays + TissUUmaps export |
| [Reports](design/11-reports.md) | Interactive HTML reports, experiment/runs index, export |
| [Configuration](design/10-configuration.md) | Draft Snakemake config files (base + per stage) |
| [Open Questions & Decisions](design/09-open-questions.md) | Resolved defaults and undecided items |
| [Appendix · Decisions](design/12-appendix.md) | Design rationale — the "why" behind the spec |

Detailed file-format specs (BEAM, embeddings, outlines) live under [formats/](formats/beam.md).
