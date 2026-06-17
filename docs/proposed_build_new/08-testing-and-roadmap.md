# 08 · Testing & roadmap

How the spec's acceptance criteria become an automated suite, and the order to build the
pipeline so each phase is validated before the next leans on it.

## Test layout

Tests live **with their component** (so they version with the code), plus shared fixtures and a
top-level cross-component integration suite:

```text
components/shared/tests/          # ids round-trip, cache_key, paths, schemas, formats, provenance
components/<stage>/tests/         # that stage's unit + contract tests
tests/
├─ integration/                   # cross-component end-to-end through Snakemake on the fixture
└─ fixtures/                      # shared fixture dataset (installed as a tiny test-only package
   ├─ mini_dataset/               #   or a path fixture both component & integration tests import)
   │                              # 2 datasets × few patients × HE/Ki67/PSA, downscaled synthetic WSIs
   ├─ manifest.yaml               # hand-written, exercises metadata at every level
   └─ labels/*.csv                # ki67 quartiles, gleason, lengths — with deliberate gaps
```

- **Per-component tests** run in that component's environment only (`uv run --package
  histomil-training pytest components/training`), so a stage's suite needs only its own deps.
- **Contract tests** for a format live in `components/shared/tests` (the contract is shared) and
  are re-exercised by the producing stage's tests against real output.
- The **fixture dataset** is the backbone: tiny synthetic OME-TIFF/NDPI slides small enough to
  run the whole DAG in CI seconds, plus deliberate edge cases (a patient missing a stain, an
  unlabelled biopsy, a label row with no matching biopsy).

## Acceptance criteria → contract tests

Each spec page's **acceptance criteria** map one-to-one onto `tests/contract/`. The schemas in
[06](06-formats-and-schemas.md) make most of them a few lines, because the contract is already
executable.

| Spec criterion | Contract test |
|---|---|
| Manifest validator fails on missing path / unknown stain / orphan label / missing HE when registering | `test_manifest_validation` |
| Folder vs flat layout both pass if manifest resolves | `test_layout_agnostic` |
| Rigid transform on raw outline ≈ rigid outline; intersection ⊆ elastic outline | `test_transform_invariants` |
| `variance_ratio ≥ 0.9` flagged; quartile cuts monotonic & equal | `test_axis_invariants` |
| Changing `overlap` only adds coords; shared positions reuse cache (no re-embed) | `test_cache_delta_fill` |
| Rebuild bundle from same cohort+cache is a no-op (identical manifest + symlinks) | `test_bundle_idempotent` |
| No fitted statistics anywhere in a bundle | `test_bundle_no_fitted_state` |
| `R·F·M` runs → `R·F·M` records, one checkpoint per fold | `test_sweep_cardinality` |
| `runs.parquet` round-trips (one run dir ↔ one row) | `test_runs_roundtrip` |
| Holdout in no fold; no patient spans train/val/test | `test_fold_leakage` |
| `development` patient scored by **exactly** its out-of-fold checkpoint | `test_oof_routing` |
| Holdout BEAM: no contributing checkpoint saw the patient | `test_holdout_routing` |
| `/attention` present iff attention architecture | `test_beam_attention_iff` |
| BEAM `patches/coords` == bundle coords, same order | `test_beam_coords_match` |
| `membership_hash` change ⇒ stale-splits warning | `test_membership_drift` |

Plus property tests: `parse(build(id)) == id` for every id type; `cache_key` stable under row
reordering; `membership_hash` invariant to input row order.

A dedicated **leakage test module** (in `components/training/tests` + `components/evaluation/tests`)
is the highest-value guard — it encodes the design's central safety claims (patient-level
splitting, locked holdout, no fitted state, OOF routing) and should fail loudly if any wiring
regresses.

## CI

- `ruff` + `mypy` (strict on `histomil-shared`) across the workspace on every push.
- **Per-component `pytest`** on every push, each in its own environment — and only the components
  a change touches need rerun, which is the payoff of the package split. `histomil-shared` tests
  need no GPU/VALIS (light deps), so they're the fast gate.
- `tests/integration` (Snakemake on the fixture, CPU embedding via `tenpercent_resnet18`) nightly
  or on demand — installs the full workspace but needs no gated weights.

## Build roadmap (phased)

Ordered so every phase is runnable and validated before the next depends on it. Each phase ends
with its contract tests green.

**Phase 0 · Workspace + shared.** The uv workspace root + `histomil-shared` built out first
(config layering, ids, paths, hashing, provenance, logging, `io`, `schemas`, `formats`, the
`report` toolkit, and `models` definitions), each stage package created as an empty shell with
its `pyproject` + `cli` scaffold, the fixture dataset, CI. No stage logic yet — but the shared
contract tests (ids/paths/schemas round-trip, cache_key, membership_hash) pass. Building `shared`
first matters: every later phase depends on it, and pinning the contracts up front is what lets
the stage packages then move independently.

**Phase 1 · Ingestion contract + pre-model reports.** `schemas/manifest` + `manifest/loader`,
`preprocess/cohort` (resolve + freeze + hash + validate), `preprocess/labels`, and the
[report toolkit](07-reports.md) with the three **pre-model pages**. This is the design's
recommended first deliverable — it validates cohorts and splits (leakage visually auditable)
with zero heavy compute. `folds.py` lands here too so the splits page is real.

**Phase 2 · WSI Transformation.** `histomil-wsi-transformation` wrapping VALIS — register + outlines + intersection
+ QC + biopsy axis. Needs the `register` extras (native libvips) so it's the first SIF-dependent
phase. Validate against `test_transform_invariants` / `test_axis_invariants`.

**Phase 3 · Embedding + bundles.** Port the embedding engine + model registry, build
`preprocess/patching`, `cache` (content-addressed, delta-fill), and `bundle`. The cache and
bundle-idempotency tests are the gate. End state: a runnable bundle.

**Phase 4 · Training.** `models/mil` (port CLAM, adapt non_clam, **build the regression head**),
`bagstore`, `trainer`, `balancing`, `aggregate` → `runs.parquet`. Then the seed-sweep cardinality
+ leakage tests, and the post-model experiment report.

**Phase 5 · Evaluation + BEAM.** `routing`, `infer` (load-once), `beam` writer, per-biopsy
evaluation reports. Routing + BEAM-invariant tests are the gate.

**Phase 6 · Heatmaps.** `warp` (raw→variant), `render`, `geojson` for TissUUmaps. Mostly ported
rendering; the new piece is correct coordinate warping.

**Phase 7 · HPO + hardening.** Optuna study + top-N promotion + segregated HPO report; SLURM
profile tuning, job grouping, weight prefetch, production SIF promotion.

> Phases 2–6 are the six stages in dependency order; 0–1 front-load the shared library and the
> cheap, high-signal cohort/split validation the docs single out as the natural starting point.

## Open items to pin before/while building

Carried from [`docs/design/09-open-questions.md`](../design/09-open-questions.md) — track
these so the build doesn't harden a guess:

- **Exact bundle manifest schema** — proposed in [06](06-formats-and-schemas.md#tabular-schemas-pandera);
  confirm before Phase 3 ends.
- **Holdout ensembling policy** — default mean; `evaluate/aggregate.py` keeps it pluggable
  (median/best-fold/per-checkpoint) until decided.
- **Combining input sources for CLAM** — affects the `models/mil/clam.py` adapter; keep the
  bag-assembly seam explicit.
