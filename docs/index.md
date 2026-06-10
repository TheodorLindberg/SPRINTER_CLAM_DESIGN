# Histology Training & Evaluation Pipeline

Design documentation for a pipeline that turns histology **whole-slide images** into trained, evaluated **MIL** models — with reproducible cohorts, embeddings, reports, and heatmaps.

> Status: **working draft.** The docs are the source of truth for the design.

## New here? Start with these

1. **[Overview](design/01-overview.md)** — what the pipeline does, in six stages.
2. **[Glossary](design/glossary.md)** — every term and abbreviation in one place.

Then walk the pipeline stage by stage under **Pipeline** in the sidebar.

## Pick a reading path

| You want to… | Read |
|---|---|
| **Understand the design** | [Overview](design/01-overview.md) → [Data Model](design/02-data-model.md) → the six **Pipeline** stages → [Reports](design/11-reports.md) |
| **Build it** | each stage's **Specification** (exact contracts) and **Implementation** (algorithms + libraries), tied together by the [Snakemake workflow](impl/workflow.md) |
| **Configure / run it** | **Configuration** (the YAML files) · [Environment & Tooling](environment.md) |
| **Know the file formats** | **Formats** ([BEAM](formats/beam.md), embeddings, outlines) |
| **See why it's designed this way** | [Appendix · Decisions](design/12-appendix.md) · [Open Questions](design/09-open-questions.md) |

## Three depth layers

Every stage is documented at three depths, so you can stay shallow or drill down:

| Layer | Answers | For |
|---|---|---|
| **Pipeline** (overview) | *what & why* | anyone |
| **Specification** | *exact contracts* — schemas, invariants, acceptance | implementers, testers |
| **Implementation** | *how* — algorithms, libraries, pseudocode | developers |

Hosted at <https://theodorlindberg.github.io/SPRINTER_CLAM_DESIGN/>.
