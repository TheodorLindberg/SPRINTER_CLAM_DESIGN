# 12 · Simplification plan — the adopted cuts

Decision taken: five sources of complexity are **removed**. This page is the concrete plan — for
each cut, what comes out, what fills the gap, the modules/configs/deps affected, and what (if
anything) we lose. It ends in the single **simplified target architecture** that supersedes the
heavier parts of [01–08].

> Nothing is built yet, so "remove" means **build the lean version from the start**, not refactor
> existing code. Where this page conflicts with [01](01-repository-layout.md)–[08](08-testing-and-roadmap.md),
> **this page wins**; those pages describe the original fuller design and the simplification
> deep-dives that motivated these cuts are [09](09-simplification-analysis.md)–[11](11-configuration-analysis.md).

## The five cuts at a glance

| # | Removed | Replaced with | Net effect |
|---|---|---|---|
| 1 | 7-package uv workspace | **One package** + `import-linter` boundaries + extras | one `pyproject`, one CLI, no star gymnastics |
| 2 | pandera + pydantic dual schemas | **Thin pydantic for config/manifest** + one `checks.py` for tables | one validation idea, one fewer dep |
| 3 | Per-row content-addressed cache + delta-fill | **File-level cache** (path = key; Snakemake is the cache) | a module + a directory + a hazard deleted |
| 4 | Augmentation as own embedding sets | **Nothing now** (off by default; a future preprocessing step) | no aug axis in cache/bundle/training |
| 5 | 8+ per-stage config files | **~5 files in a top-level `config/`**, by cadence; selection on CLI | one edit surface, fewer files |

The cuts reinforce each other: dropping the workspace (1) is what lets configs consolidate into one
tree (5) and removes the report-toolkit contortion; dropping augmentation (4) shrinks the cache key
that (3) already simplifies.

---

## Cut 1 · One package instead of the workspace

**Remove.** The `components/*` workspace, the 7 `pyproject.toml`s, workspace sources, per-component
console scripts, and the hand-policed "star."

**Replace with.** A single installable `histomil` package whose submodules are the stages, with the
no-cross-stage-import rule enforced **mechanically** by `import-linter` in CI.

```text
src/histomil/
  shared/            # foundation: config, ids, paths, io, formats, models, provenance, report toolkit
  wsi_transformation/  preprocessing/  training/  evaluation/  heatmaps/  reporting/
  cli.py             # one Typer app: `histomil <stage> <cmd>`
```

**Concrete changes.**

- **One `pyproject.toml`** at the root; heavy capabilities are **extras** (`histomil[wsi]`,
  `[register]`, `[embed]`, `[train]`, `[reports]`). The SIF installs `histomil[all]`.
- **One CLI** (`histomil preprocess cohort`, `histomil train run`, …) replaces six scripts.
- **`import-linter` contract** (CI): `histomil.<stage>` may import `histomil.shared`, never another
  stage. This *is* the star, now enforced by a tool instead of vigilance.
- Snakemake rules return to `workflow/rules/{stage}.smk`; rule bodies call `histomil <stage> <cmd>`.

**What fills the gaps.**

- *Enforced boundaries* → `import-linter` (stronger than the manual rule it replaces).
- *Independent install* (rarely used) → extras give a torch-free reporting env etc. without
  separate packages.
- *The report-toolkit-in-`shared` contortion from [07] disappears*: with one package there are no
  inter-package deps, so plotly/jinja2 are just an extra; the toolkit can stay in `shared.report`
  at zero cost, and `preprocessing` emitting the cohort report is a normal intra-package import.

**What we lose.** Independent *distribution/versioning* of a stage — not used in this usecase.
**Trigger to revisit:** an external consumer or a hard dependency conflict between stages; splitting
a clean single package (boundaries already enforced) into a workspace later is mechanical.

---

## Cut 2 · One lightweight validation idea, not two schema systems

**Remove.** `pandera` entirely, and the notion of declarative dataframe schemas.

**Replace with.** Two small, explicit things:

- **Thin `pydantic` kept for `config` + `manifest` only.** This is itself a simplification: it
  gives validated, typed config with `extra='forbid'` (catches typo'd keys) and numeric coercion
  (kills the `1e-4`-parses-as-string trap from [11](11-configuration-analysis.md)) in a few model
  classes — far less code than hand-rolled parsing, and it's the one place validation clearly pays.
- **One `histomil/shared/checks.py`** of explicit functions for the handful of *leakage-critical*
  table invariants: `check_no_fold_leakage(df)`, `check_holdout_excluded(folds, membership)`,
  `check_bundle_manifest(df)`, `check_membership(df)`, `check_coords_match(embeddings, coords)`. A
  plain `pandas` assertion is clearer than a schema declaration for these cross-row rules anyway —
  which was pandera's only real value here ([09](09-simplification-analysis.md)).

**Concrete changes.**

- `shared/schemas/` (the pandera-heavy module set in [06](06-formats-and-schemas.md)) collapses to:
  `shared/config.py` (pydantic config models), `shared/manifest.py` (pydantic manifest model), and
  `shared/checks.py` (assertion functions). Column dtypes are enforced on read by a small
  `read_table(path, columns={...})` helper, not a schema object.
- `pandera` leaves the dependency list.

**What fills the gaps.**

- *Dataframe contracts* → `checks.py`, called by the producing rule **and** the contract tests
  (same function, two callers — [08](08-testing-and-roadmap.md)'s acceptance tests still map 1:1).
- *Config/manifest validation* → the thin pydantic models.

**Alternative if you want `pydantic` gone too.** Use `@dataclass` + `yaml.safe_load` + a manual
`validate()` in `config.py`. Honest tradeoff: you re-implement coercion and the `1e-4` guard by
hand and lose typed errors — *more* code and footguns for one fewer dependency. **Recommendation:
keep the thin pydantic; it reduces complexity rather than adding it.** This is the one cut on the
list I'd apply partially.

---

## Cut 3 · File-level embedding cache

**Remove.** The per-row content-addressed store, `delta-fill`, the canonical-row-order requirement,
and the coords-freshness coupling. (Full argument: [10](10-embedding-cache-analysis.md).)

**Replace with.** The embedding file **path encodes the full config**; Snakemake's file existence is
the cache.

```text
processed/embeddings/{model}/{scan}__{variant}__{patch_config}.h5
```

**Concrete changes.**

- Delete `preprocessing/cache.py` and `paths.cache_object`. `embed` becomes an ordinary rule
  (`input: coords + wsi → output: the per-config h5`), writing rows **once, in coordinate order**.
- Bundles **symlink the DAG-tracked files directly**.
- Optionally mark `embed` cacheable (`snakemake --cache`) for free cross-run dedup.
- [04](04-snakemake-integration.md)'s entire "Embedding cache vs. the DAG" section is deleted.

**What fills the gaps.** Cross-cohort reuse is already free (cohort isn't in the path). *Cross-overlap
sub-file reuse* is lost — accepted, since the usecase fixes one patch config. **Trigger:** frequent
overlap/patch-size sweeps → adopt fine-grid-then-decimate (still no row store).

**What we lose.** Re-embedding shared positions *if* you sweep overlap — a sweep not currently run.
**What we gain:** a silent attention↔coordinate misalignment hazard is removed structurally.

---

## Cut 4 · No augmentation embedding sets

**Remove.** The `embedding.augmentation` config block, the `augmentation_id` cache-key axis, the
`use_augmented_embeddings` training flag, the `augmented`-tagged bags, and the train-fold-only
augmented-bag logic.

**Replace with.** Nothing, for now. Each patch is embedded **once**; the foundation model runs a
single forward per patch.

**Concrete changes.**

- `preprocessing/embed.py` loses the augmentation loop; the cache key loses `aug`; the embedding
  path loses the `{aug}` segment.
- `bundle` manifest loses the `augmented` tag; `training` loses augmented-bag handling and the
  config flag; `seeds`/balancing logic simplifies.

**What fills the gaps.** If augmentation is ever wanted, it returns as a **preprocessing step**
(the design's placement was correct) — built then, behind a flag, and measured. Foundation-model
embeddings are largely augmentation-robust, so this is deferred until a held-out uplift is shown
([09](09-simplification-analysis.md)).

**What we lose.** Possible augmentation uplift — converted from infrastructure into a future
experiment.

---

## Cut 5 · One config tree, by cadence

**Remove.** The 10 scattered per-stage config files (and their per-component homes from
[01](01-repository-layout.md)).

**Replace with.** A single top-level `config/`, organized by **edit cadence**
([11](11-configuration-analysis.md)), with stage configs merged into sections and one file per
experiment:

```text
config/
  base.yaml            # infra: roots, backends, embedding registry, reference_stain, eval/heatmap defaults
  cohorts.yaml         # APPEND-ONLY registry (version with _vN)
  seeds.yaml           # APPEND-ONLY registry
  pipeline.yaml        # wsi_transformation + preprocessing + reports — as sections (occasional defaults)
  experiments/
    ki67_stain_comparison.yaml   # one file per experiment: defaults + runs (+ optional hpo block)
```

~5 files instead of 10. `evaluation.yaml` and `heatmaps.yaml` as standalone files are gone:
their *defaults* live in `base.yaml`, and the *selection* (which model/subset/figure) moves to the
**CLI** (`histomil evaluate --experiment X --subset holdout`).

**Concrete changes.**

- Configs leave `components/*/config/` for the single `config/` tree.
- `shared.config` layers `base.yaml` → `pipeline.yaml` (or an `experiments/*.yaml`) → `--set` CLI
  overrides; each section validates against its pydantic model.
- Runs reference bundles by **components**, not hand-typed `bundle_id`s; ids/outputs are derived
  ([11 · B](11-configuration-analysis.md)).
- A fast `histomil config validate` checks cross-file references + membership-hash drift before any
  compute; the resolved config is snapshotted into each run's output dir.

**What fills the gaps.**

- *"Run one stage" ergonomics* → preserved: `base.yaml` always loads, `pipeline.yaml` sections stay
  separable, and a stage target reads only what it needs.
- *Per-run selection* → CLI targets + `--set`, so files aren't edited-and-reverted.

**What we lose.** Nothing structural — the cohort/seed/bundle/experiment model is unchanged; only
the files' homes and granularity change.

---

## The resulting simplified architecture

```text
histo-mil/
├─ pyproject.toml                 # ONE package; extras: wsi, register, embed, train, reports, all
├─ uv.lock
├─ config/                        # base.yaml · cohorts.yaml · seeds.yaml · pipeline.yaml · experiments/*.yaml
├─ src/histomil/
│  ├─ shared/                     # config, ids, paths, io, formats(beam,outline), models(embedding,mil),
│  │                              #   provenance, checks.py, report toolkit
│  ├─ wsi_transformation/  preprocessing/  training/  evaluation/  heatmaps/  reporting/
│  └─ cli.py                      # one Typer app
├─ workflow/                      # Snakefile + rules/{stage}.smk + profiles/slurm
├─ ingestion/                     # stage-1 bridges (unchanged)
├─ containers/  scripts/
└─ tests/                         # unit + contract (checks.py) + integration; .importlinter contract
```

**Dependencies (one set, grouped by extra):** core = `pydantic, pandas, pyarrow, numpy, h5py,
pyyaml, shapely, typer, structlog, gitpython`; `wsi/register/embed/train/reports` extras as before;
**dropped:** `pandera`, and the per-component packaging. **Added:** `import-linter` (dev).

**Validation:** pydantic for config + manifest; `shared/checks.py` for the leakage-critical tables;
`read_table` for dtypes. **Cache:** file-level, path = key. **Augmentation:** none.

## Impact on the existing build-plan pages

| Page | Change |
|---|---|
| [01 Repository layout](01-repository-layout.md) | Replaced by the single-package tree above |
| [02 Environment & packages](02-environment-and-packages.md) | One `pyproject` + extras; drop workspace + `pandera`; add `import-linter` |
| [03 Shared package](03-shared-core.md) | Now a submodule `histomil.shared`; `schemas/` → `config.py` + `manifest.py` + `checks.py`; toolkit stays, cost-free |
| [04 Snakemake](04-snakemake-integration.md) | Rules in `workflow/rules/`; **delete** the cache-vs-DAG section; rule bodies call `histomil <stage>` |
| [05 Stage impls](05-stage-implementations.md) | Drop `cache.py` + augmentation; module paths are submodules |
| [06 Formats & schemas](06-formats-and-schemas.md) | pandera schemas → `checks.py` + `read_table`; embedding path/key lose `aug`; layouts otherwise unchanged |
| [07 Reports](07-reports.md) | The star-constraint workaround is moot; toolkit in `shared.report` is now ordinary |
| [08 Testing & roadmap](08-testing-and-roadmap.md) | Per-component test split → one suite + `import-linter` check; Phase 0 builds `shared` in one package |

## Net complexity reduction

- **−6 `pyproject.toml`**, −6 console scripts, −1 packaging concept (workspace) → 1 package, 1 CLI.
- **−1 dependency** (`pandera`); validation concentrated in `config.py` + `manifest.py` +
  `checks.py`.
- **−1 module** (`cache.py`), **−1 directory** (per-position store), **−1 page** of Snakemake
  caveats, **−1 silent-correctness hazard**.
- **−1 cache axis + config block + training flag** (augmentation).
- **10 → ~5 config files**, one edit surface, selection on the CLI.

Everything that protects the science is untouched: cohorts/roles/splits + `membership_hash`, OOF
routing, provenance, raw/rigid/elastic, BEAM, the seed sweep. The simplification is entirely on the
engineering-flexibility side, which is exactly where it should fall.

## Revised first steps

1. Scaffold the **one package** + `config/` + `import-linter` contract + CI (was Phase 0).
2. `shared`: `config.py`/`manifest.py` (pydantic), `ids`, `paths`, `io`, `checks.py`, `formats`.
3. Cohort resolution + pre-model reports (unchanged value, now no packaging overhead).
4. Proceed through the stages as in [08](08-testing-and-roadmap.md), minus `cache.py` and
   augmentation, with the file-level embedding cache.
