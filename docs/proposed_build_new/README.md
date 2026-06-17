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
| [03 · The shared subpackage](03-shared-core.md) | `histomil.shared`: config layering, id/path helpers, IO, formats, checks, model definitions, provenance — the foundation every stage imports. |
| [04 · Snakemake integration](04-snakemake-integration.md) | Snakefile structure, rule modules, wildcards, checkpoints, the SLURM profile, job grouping, how rules call the package. |
| [05 · Stage implementations](05-stage-implementations.md) | Module-by-module breakdown of stages 2–6 + reports: the functions, their inputs/outputs, and the third-party engines each wraps. |
| [06 · Formats & schemas](06-formats-and-schemas.md) | Every artifact (HDF5, CSV/Parquet, JSON, GeoJSON) as a concrete schema, mapped to the format specs. |
| [07 · Reports toolkit](07-reports.md) | The static-HTML + Plotly report builder, `runs.parquet`, faceted indexes, export. |
| [08 · Testing & roadmap](08-testing-and-roadmap.md) | Test layout, how acceptance criteria become tests, and the phased build order. |

## Guiding principles for the build

These fall directly out of the design and shape every page that follows:

1. **One package, enforced internal boundaries.** The repo is a single installable package
   (`histomil`) whose top-level submodules are the six stages plus reporting, sharing a
   `histomil.shared` foundation. An `import-linter` contract in CI enforces that a stage may
   import `shared` but never a sibling stage — so stages evolve independently without separate
   distributions. Snakemake rules and one Typer CLI (`histomil <stage> <cmd>`) are the only entry
   points. Stage 1 (ingestion) is the one exception — a user-written bridge against a published
   contract.
2. **The manifest is the contract, not the filesystem.** Code resolves scans through the
   [scan manifest](../spec/data-ingestion.md), never by walking directories. Paths are
   computed from `roots` in `base.yaml` by one path resolver, never hard-coded.
3. **Contracts are executable, lightly.** Config and the manifest are `pydantic` models;
   leakage-critical tables are explicit assertion functions in `shared.checks`; every binary file
   has a typed reader/writer. Each contract is one definition, reused by the pipeline and the
   test suite.
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

- One package, `histomil`, with stage submodules: `histomil.wsi_transformation`,
  `histomil.preprocessing`, `histomil.training`, `histomil.evaluation`, `histomil.heatmaps`,
  `histomil.reporting`, on the `histomil.shared` foundation.
- Heavy capabilities are **extras** (`histomil[wsi]`, `[register]`, `[embed]`, `[train]`,
  `[reports]`, `[all]`), so a light environment installs only what it needs.
- One Typer CLI with stage subcommands (`histomil preprocess …`, `histomil train …`).

The import graph is a **star**: every stage → `histomil.shared`, no stage → another, enforced by
`import-linter`. This is what makes "updated independently" hold without separate distributions.
