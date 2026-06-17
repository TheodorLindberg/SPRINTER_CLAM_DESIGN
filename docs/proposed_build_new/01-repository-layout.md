# 01 · Repository layout

The build is **one installable Python package** (`histomil`) whose top-level submodules are the
stages, sharing a `histomil.shared` foundation. Stage-to-stage import boundaries are enforced
mechanically by `import-linter` (a stage may import `shared`, never a sibling stage). A thin
`workflow/` wires the stages into one Snakemake DAG. Configs live in a single top-level `config/`
tree (user-facing data, not code). Nothing computes from the directory tree
([principle 2](README.md#guiding-principles-for-the-build)); this layout is for humans and packaging.

```text
histo-mil/                              # repo root (currently `design_docs/`)
├─ pyproject.toml                       # ONE package; extras: wsi, register, embed, train, reports, all — see 02
├─ uv.lock
├─ README.md  CLAUDE.md
├─ docs/                                # the design docs (authoritative)
├─ proposed_build/                      # this plan
│
├─ src/histomil/                        # the one package
│  ├─ __init__.py
│  ├─ cli.py                            # one Typer app: `histomil <stage> <cmd>`
│  │
│  ├─ shared/                           # the foundation every stage imports (see 03)
│  │  ├─ config.py                      # pydantic models + base+stage layering loader
│  │  ├─ manifest.py                    # scan-manifest pydantic model + validator
│  │  ├─ ids.py                         # build/parse scan_id, bag_id, bundle_id
│  │  ├─ paths.py                       # roots → concrete paths (the only path authority)
│  │  ├─ checks.py                      # leakage-critical dataframe invariants (assert functions)
│  │  ├─ provenance.py  hashing.py  logging.py
│  │  ├─ io/                            # typed IO: h5.py geojson.py tables.py wsi.py
│  │  ├─ formats/                       # cross-stage binary formats: beam.py outline.py
│  │  ├─ report/                        # report toolkit: toolkit.py assets/reports.css
│  │  └─ models/                        # model DEFINITIONS shared by preprocess/train/eval
│  │     ├─ embedding/                  # registry: id → (load(pretrained), NORM)
│  │     │   __init__.py base.py uni2_h.py conch.py gigapath.py tenpercent_resnet18.py
│  │     └─ mil/                        # base.py clam.py non_clam.py regression.py
│  │
│  ├─ wsi_transformation/               # Stage 2 — register.py outlines.py intersection.py biopsy_axis.py qc.py
│  ├─ preprocessing/                    # Stage 3 — cohort.py cohort_report.py labels.py patching.py embed.py bundle.py
│  ├─ training/                         # Stage 4 — folds.py trainer.py balancing.py bagstore.py runrecord.py aggregate.py hpo.py
│  ├─ evaluation/                       # Stage 5 — routing.py infer.py aggregate.py
│  ├─ heatmaps/                         # Stage 6 — warp.py render.py geojson.py
│  └─ reporting/                        # datasets.py splits.py experiments.py hpo.py evaluation.py index.py
│
├─ workflow/                            # thin orchestration only (see 04)
│  ├─ Snakefile                         # includes rules/*.smk; defines `rule all`
│  ├─ rules/                            # common.smk + one .smk per stage
│  └─ profiles/slurm/config.yaml
│
├─ config/                              # all configs, by edit cadence (design doc 10)
│  ├─ base.yaml  cohorts.yaml  seeds.yaml
│  ├─ pipeline.yaml                     # wsi_transformation + preprocessing + reports (sections)
│  └─ experiments/{name}.yaml           # one file per experiment (defaults + runs + optional hpo)
│
├─ ingestion/                           # Stage 1 — user-written bridges (outside Snakemake)
│  ├─ README.md                         # → spec/data-ingestion.md (the contract)
│  └─ sahlgrenska_2018.py               # shipped reference ingester
│
├─ containers/                          # histomil.def, build.sh — see 02
├─ scripts/                             # run.sh (sbatch controller), prefetch_weights.py
└─ tests/                               # unit + contract (checks.py) + integration; .importlinter contract
```

## One package, enforced boundaries

There is no inter-package dependency graph to police by hand; instead an **`import-linter`
contract** in CI encodes the rule:

> `histomil.<stage>` may import `histomil.shared`; no stage module may import another stage module.

This gives the separation-of-concerns the design wants — stages evolve independently, each is
testable in isolation — without the overhead of separate distributions. The things that would
otherwise tempt one stage to import another live in `shared`:

- **BEAM read/write** in `shared.formats.beam` — so heatmaps and reporting read evaluation's output
  without importing `histomil.evaluation`.
- **MIL heads** in `shared.models.mil` — so evaluation reconstructs a checkpoint's architecture
  without importing `histomil.training`.
- **Schemas + the embedding registry + the WSI reader** in `shared` — so training/evaluation read
  what preprocessing wrote against one contract.
- **The report toolkit** in `shared.report` — so `preprocessing.cohort_report` emits the cohort
  page with the same macros `reporting` uses, no cross-stage import.

Rule of thumb: **definitions and contracts are shared; behaviour is owned by the stage.**

## What goes in `shared` vs a stage

| Belongs in `histomil.shared` | Belongs in a stage submodule |
|---|---|
| Config models + the base-layering loader; the manifest model | The stage's own config-section model |
| Id/path construction, hashing, provenance | — |
| Typed IO (HDF5, GeoJSON, tables, WSI reader) | — |
| Binary **format** read/write (BEAM, outline) | The logic that *fills* a format |
| Model **definitions** (embedding registry, MIL heads) | Training loop, inference loop |
| Report **toolkit** (macros, CSS, provenance block) | *Which* pages exist + *what* they plot |
| `checks.py` (leakage-critical invariants) | The stage's algorithm + its own tests |

## Configs live in `config/`, not with the code

Configs are user-facing data edited constantly, so they sit in one top-level `config/` tree
organized by edit cadence — not scattered beside the code that reads them. Loading always layers
`config/base.yaml` first, then the file for what's running (`pipeline.yaml` or an
`experiments/<name>.yaml`), then any `--set` CLI overrides. The config model lives in
`shared.config`; see the [configuration design doc](../design/10-configuration.md).

## The CLI surface (one app, stage subcommands)

`histomil/cli.py` is a single Typer app; each Snakemake rule is a one-line call to a subcommand, and
every stage is independently runnable by hand:

```text
histomil wsi register | biopsy-axis
histomil preprocess cohort | derive-labels | coords | embed | bundle
histomil train folds | run | aggregate | hpo
histomil evaluate infer | beam
histomil heatmap render
histomil report build --section ...
```

Each command parses configs via `histomil.shared.config`, resolves paths via
`histomil.shared.paths`, and delegates to its stage submodule — never the reverse, and never into a
sibling stage.
