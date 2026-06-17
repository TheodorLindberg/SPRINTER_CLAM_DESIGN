# Proposed build plan

A concrete engineering plan for turning the design docs in [`docs/`](../index.md) into a
running codebase. Where `docs/` answers **what** the pipeline must do and **why**,
these pages answer **how to build it**: the repository layout, the Python package
boundaries, the shared library every stage leans on, the exact on-disk formats as
code-level schemas, the Snakemake wiring, the dependency set, and a phased roadmap.

> The design docs remain the **authoritative source of truth**. Nothing here may
> contradict a [spec](../spec/index.md) contract or an [appendix](../design/12-appendix.md)
> decision. Where this plan makes a choice the docs left open (library, module
> boundary, signature), that choice is flagged as **[build choice]** and is changeable
> as long as the spec still holds.

## Reading order

| Page | What it pins down |
|---|---|
| [01 · Repository layout](01-repository-layout.md) | The full directory tree: the `histomil` package, the Snakemake workflow, configs, containers, tests. |
| [02 · Environment & packages](02-environment-and-packages.md) | Python version, `uv`/`pyproject`, the dependency set grouped by concern, the SIF build, env vars. |
| [03 · The shared package](03-shared-core.md) | `histomil-shared`: config layering, id/path helpers, IO, schemas, formats, model definitions, provenance — the one package every stage imports. |
| [04 · Snakemake integration](04-snakemake-integration.md) | Snakefile structure, rule modules, wildcards, checkpoints, the SLURM profile, job grouping, how rules call the package. |
| [05 · Stage implementations](05-stage-implementations.md) | Module-by-module breakdown of stages 2–6 + reports: the functions, their inputs/outputs, and the third-party engines each wraps. |
| [06 · Formats & schemas](06-formats-and-schemas.md) | Every artifact (HDF5, CSV/Parquet, JSON, GeoJSON) as a concrete schema, mapped to the format specs. |
| [07 · Reports toolkit](07-reports.md) | The static-HTML + Plotly report builder, `runs.parquet`, faceted indexes, export. |
| [08 · Testing & roadmap](08-testing-and-roadmap.md) | Test layout, how acceptance criteria become tests, and the phased build order. |
| [09 · Simplification analysis](09-simplification-analysis.md) | Strict critical review: what to cut/downgrade/keep, the tradeoffs, and what complexity genuinely protects the science. |
| [10 · Embedding cache](10-embedding-cache-analysis.md) | Focused deep-dive: drop the content-addressed store for a file-level "Snakemake is the cache" scheme; the row-vs-file impedance mismatch and its hazards. |
| [11 · Configuration](11-configuration-analysis.md) | How configs are written, stored, and edited in daily use; separating selection from definition, deriving ids, top-level layout, early validation. |

## Guiding principles for the build

These fall directly out of the design and shape every page that follows:

1. **A uv workspace of independent stage packages.** The repo is a [uv
   workspace](https://docs.astral.sh/uv/concepts/projects/workspaces/); each of the six stages
   plus reporting is its **own installable distribution** (`histomil-preprocessing`,
   `histomil-training`, …) under the shared `histomil.` namespace, with its own `pyproject`,
   its own configs, and its own Snakemake rules — so each is versioned, tested, and updated
   independently. They depend on exactly one common package, `histomil-shared`, and never on
   each other. Snakemake rules and each package's thin CLI are the only entry points. Stage 1
   (ingestion) is the one exception — a user-written bridge against a published contract.
2. **The manifest is the contract, not the filesystem.** Code resolves scans through the
   [scan manifest](../spec/data-ingestion.md), never by walking directories. Paths are
   computed from `roots` in `base.yaml` by one path resolver, never hard-coded.
3. **Schemas are executable.** Every manifest/label/fold table has a `pandera` schema; every
   config has a `pydantic` model; every binary file has a typed reader/writer. Validation is a
   library function, reused by both the pipeline and the test suite.
4. **No fitted state escapes training.** The bundle layer physically cannot write normalization
   stats (the writer has no field for them); fitting happens only inside the per-fold trainer.
5. **Reuse the proven engines, replace the glue.** Port the old pipeline's embedding engine,
   model registry, patch extraction, and SLURM wiring (see `OLD_PIPELINE_REVIEW.md` §3, at the
   repo root outside the docs site); rebuild identity, bundling,
   splits, results, and config as designed.
6. **Provenance on every artifact.** A shared provenance block (git commit, config hash,
   membership hash, timestamps) is stamped into run records, BEAM attrs, and report pages by
   one helper.

## Naming convention

*(build choice — "histology MIL"; rename is a find-replace of the `histomil` prefix.)*

- The repo is a **uv workspace**. Each component is a distribution named
  `histomil-<component>` that provides the import namespace `histomil.<component>` (PEP 420
  implicit namespace packages — no top-level `__init__.py`).
- The common code is `histomil-shared` → `histomil.shared`.
- Components: `histomil-wsi-transformation`, `histomil-preprocessing`, `histomil-training`,
  `histomil-evaluation`, `histomil-heatmaps`, `histomil-reporting`, and `histomil-shared`.
- Each component ships a console script (`histomil-preprocess`, `histomil-train`, …); there is
  no monolithic CLI, so importing one stage never drags in another's heavy deps.

The dependency graph is a **star**: every component → `histomil-shared`, and no component →
another. This is what makes "updated independently" hold.
