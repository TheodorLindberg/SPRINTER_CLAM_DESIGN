# 11 · Configuration — how it's written, stored, and edited

A focused review of the config system, judged by the question that actually matters in daily use:
**who edits which file, how often, and in what way?** Structure that ignores edit-cadence looks
tidy on paper and fights you at the keyboard. The current scheme
([`docs/design/10-configuration.md`](../design/10-configuration.md), the drafts under
[`docs/configs/`](../configs/preprocessing.md)) is reasonable but mixes *definitions*,
*selections*, and *derived values* in the same files and scatters them across component
directories. This page names the specific frictions and proposes how to store and shape configs so
editing matches how the application is run.

Companion to [09 · Simplification analysis](09-simplification-analysis.md) and
[01 · Repository layout](01-repository-layout.md).

## The editing reality (classify by cadence, not by stage)

The configs are not one kind of thing. They differ by *who touches them and how often* — and that,
not which stage consumes them, should drive where they live and how they're shaped:

| Config | Role | Edit cadence | Edit pattern |
|---|---|---|---|
| `base.yaml` | roots, registry paths, backends | **set once** per deployment | modify rarely |
| `cohorts.yaml` | named patient groups + holdout | **append-mostly** | **add** entries; *editing an existing one is dangerous* (see below) |
| `seeds.yaml` | named split sets | **append-mostly** | add entries; same freezing hazard |
| `wsi_transformation.yaml` | registration + outline options | occasional | modify when method/data changes |
| `preprocessing.yaml` | labels, patching, embedding, bundle | occasional | modify when labels/patch/embedder change |
| `model_experiment.yaml` | defaults + runs | **frequent** | add/modify runs constantly |
| `hpo.yaml` | search space | per search | new file per search |
| `evaluation.yaml` | which model, subset, output | **per run** | re-point at a different model each time |
| `heatmaps.yaml` | variant, colormap, render | per visual | tweak per figure |
| `reports.yaml` | sections, export | set once | rarely |

Three tiers fall out: **infrastructure** (set once: `base`), **registries** (append-only, frozen:
`cohorts`, `seeds`), **stage defaults** (occasional: `wsi_transformation`, `preprocessing`,
`reports`), and **experiment/run** (frequent: `model_experiment`, `hpo`, `evaluation`,
`heatmaps`). The frequent tier is where ergonomics matter most — and it's where the current scheme
has the most friction.

## Specific frictions in the current scheme

### 1 · The edit surface is scattered across component directories

[01](01-repository-layout.md) places each stage's config in `components/<name>/config/`, with
`base`/`cohorts`/`seeds` in `shared`. So a user running an experiment edits
`components/training/config/model_experiment.yaml`, points it at a cohort in
`components/shared/config/cohorts.yaml`, and evaluates via
`components/evaluation/config/evaluation.yaml` — three component trees for one logical task.
**Configs are user-facing data, not code**, yet the packaging split filed them by which code reads
them. (This is another cost of the workspace split flagged in [09](09-simplification-analysis.md).)

### 2 · Per-invocation *selections* are baked in as config

Several files hard-code "what am I running right now," which is really a **target**, not a setting:

- `wsi_transformation.yaml` pins `dataset_id: sahlgrenska_2018` and `input_manifest: …`. Processing
  another dataset means editing the file (or cloning it).
- `evaluation.yaml` pins `bundle:`, `model:`, `subset:`, and a hand-written
  `output_dir: results/evaluation/he_lr1e4__holdout_v1`. Evaluating a different model = edit the
  file every time.

These are exactly the axes Snakemake already expresses as **wildcards** (`dataset`, `bundle_id`,
`run_id`, `subset`). Encoding them in config duplicates the wildcard space and forces a file edit
for what should be a command-line target.

### 3 · Derived identifiers are hand-typed

`bundle_id` is *derived* from `cohort__stain__variant__embedding`
([data model](../design/02-data-model.md#identifiers)) — yet it appears as a literal string in
`model_experiment.yaml` (`bundle: prostate_combined_v1__sHE__raw__conch`), `hpo.yaml`, and
`evaluation.yaml`. Hand-writing a derived id invites typos and **drifts** if the naming scheme ever
changes. The smell is explicit in `preprocessing.yaml`: `embedding_model: conch` appears twice with
the comment *"must match embedding.model above"* — two copies of one fact, kept in sync by a
comment. `output_dir` in `evaluation.yaml` is likewise a hand-built restatement of
`run_id + subset`.

### 4 · Registry edits silently invalidate frozen state

`cohorts.yaml` and `seeds.yaml` are **registries whose entries get frozen** (the
`membership_hash`). Editing an existing cohort's members changes the hash and **silently stales
every split and bundle built from it** ([training](../spec/training.md) warns, but only at run
time). Nothing in the config *shape* signals "this is append-only; version, don't edit." The
manual convention is the `_v1` suffix (`prostate_combined_v1`), but it's undocumented discipline,
not enforced.

### 5 · YAML + no edit-time validation

- **The scientific-notation trap:** in PyYAML, `1e-4` parses as a **string**, while `1.0e-4`
  parses as a float. The draft uses the safe form, but a user typing `lr: 1e-4` gets a silent
  string → a cryptic downstream failure. This *will* bite.
- **Unknown keys pass silently.** A typo'd key (`droput: 0.3`) is ignored; the default is used and
  the run looks fine but isn't what you meant.
- Errors surface **late** — often after a long job starts — rather than at edit time.

### 6 · The config that produced a result isn't snapshotted

A run records a `config_hash` + `git_commit` ([provenance](03-shared-core.md#provenance-hashing)).
But a hash only tells you *that* the config changed, not what it was. Since configs are hand-edited
in place, the file at HEAD may no longer match what produced last week's result — and a hash can't
reconstruct it.

## How to best adjust

### A · Store configs by cadence, in one top-level tree (not by component)

Pull configs out of the component directories into a single user-facing tree, organized by the
tiers above:

```text
config/
  base.yaml                      # infrastructure — set once
  registries/
    cohorts.yaml                 # APPEND-ONLY · version with _vN, never edit a frozen entry
    seeds.yaml                   # APPEND-ONLY
  stages/
    wsi_transformation.yaml      # occasional defaults (no dataset_id — that's a target)
    preprocessing.yaml
    reports.yaml
  experiments/
    ki67_stain_comparison.yaml   # ONE FILE PER EXPERIMENT (defaults + runs)
    psa_baseline.yaml
  hpo/
    ki67_he_search.yaml          # one file per search
```

This makes the daily edit surface a single directory, groups files by how often you touch them, and
is cleaner under *either* packaging choice — the component code still loads them via
[`shared.config`](03-shared-core.md#config-histomilsharedconfig); only their *home* moves to where the user lives.
**One file per experiment** leans into the design's "config count = O(experiments)" principle: each
experiment is self-contained, diffable, and archivable next to its results.

### B · Reference definitions; derive identifiers and outputs

Stop hand-typing derived ids. A run should name its bundle by **components or a one-time alias**,
and let [`shared.ids`](03-shared-core.md#ids-histomilsharedids) build the `bundle_id`:

```yaml
# instead of:  bundle: prostate_combined_v1__sHE__raw__conch
bundle: { cohort: prostate_combined_v1, stain: HE, variant: raw, embedding: conch }
# or a named alias defined once in preprocessing.yaml and referenced everywhere.
```

- `output_dir` should be **derived** by the path authority from `run_id + subset`, never written.
- `embedding_model` should have **one** home (the embedding block); the bundle inherits it rather
  than restating it with a "must match" comment.

If you keep string ids for brevity, at least **validate that they resolve** (the bundle's cohort
exists, its stain/variant/embedding are real) at load time.

### C · Separate *selection* from *definition*: targets on the CLI, definitions in files

Configs hold **definitions and defaults**; *what you're running now* is a **target**:

```bash
histomil-wsi register   --dataset sahlgrenska_2018          # manifest path derived from roots+dataset
histomil-evaluate       --run he_lr1e4 --subset holdout     # reads the run from runs.parquet; output derived
histomil-train          --experiment ki67_stain_comparison  # the experiment file = the definition
```

For one-off experimentation, support **ad-hoc CLI overrides** layered last
(`base < stage < --set`), so a quick test doesn't mean editing and reverting a file:

```bash
histomil-train --experiment ki67_stain_comparison --set defaults.hyperparameters.epochs=5
```

This removes the "edit `evaluation.yaml` for every model" and "edit-then-revert for a quick test"
loops entirely.

### D · Validate early, cheaply, and across files

Add a fast `histomil config validate` (and run it at the head of every stage) that, with **zero
heavy compute**:

- loads every config through pydantic with **`extra='forbid'`** → unknown keys are errors, not
  silent defaults; coerces numerics so the `1e-4` trap can't survive;
- checks **cross-file references**: every run's bundle resolves; `seed_set.cohort == bundle.cohort`
  ([training requires this](../spec/training.md)); `target` exists in `preprocessing.derived`
  labels; holdout patients exist in the manifest;
- recomputes each referenced cohort's `membership_hash` and **warns if it differs** from the frozen
  value — surfacing stale splits at *edit* time, not six hours into a job.

This generalizes the cohort-resolution validation the design already does, into a pre-flight for
the whole config set.

### E · Make registries append-only by discipline *and* signal

Document and lint the rule: **never edit a frozen `cohorts`/`seeds` entry; add `name_v2`.** The
`validate` step's membership-hash check (D) is the enforcement; a short header comment in the file
and a `frozen: <hash>` field per entry make the intent explicit and machine-checkable.

### F · Snapshot the resolved config with the result

On every run, write the **fully resolved** config (base + stage + experiment + CLI overrides,
defaults expanded) into the run/experiment output dir — not just its hash. Then a result is
reproducible regardless of later edits to the source files, and `runs.parquet` can point at the
exact config that produced each row.

## Worked example: "compare CONCH vs UNI2 on Ki67 holdout"

**Current scheme.** Build a second bundle: edit `preprocessing.yaml` (`embedding.model: uni2_h`
*and* `bundle.embedding_model: uni2_h`), rerun. Add runs: edit `model_experiment.yaml`, hand-type
`bundle: prostate_combined_v1__sKi67__raw__uni2_h` (typo risk). Evaluate: edit `evaluation.yaml`'s
`bundle`, `model`, `output_dir` for the CONCH model, run; then edit all three again for UNI2.
**~5 file edits, two of them duplicated, two hand-typed derived ids.**

**Adjusted scheme.** In `config/experiments/ki67_stain_comparison.yaml`, add two runs referencing
bundles by components (`embedding: conch` / `embedding: uni2_h`); the bundle_ids are derived.
Build + train: `histomil-train --experiment ki67_stain_comparison`. Evaluate both:
`histomil-evaluate --experiment ki67_stain_comparison --subset holdout` (iterates the experiment's
runs; outputs derived). **One file edit, no hand-typed ids, no per-model evaluation edits.**

## Bottom line

| Adjustment | Removes |
|---|---|
| Top-level `config/` by cadence; one file per experiment | scattered edit surface; "which component owns this?" |
| Reference components, derive ids/outputs | hand-typed `bundle_id` typos; "must match" duplication |
| Targets/CLI for selection; `--set` overrides | editing files to pick what to run; edit-then-revert |
| `config validate` (extra=forbid + cross-refs + hash) | YAML traps, silent typos, late failures, stale splits |
| Append-only registries + `frozen:` hash | silent split invalidation |
| Snapshot resolved config with results | irreproducible results after a config edit |

None of this changes *what* the configs express — the cohort/seed/bundle/experiment model from the
design stays. It changes **where they live and how you touch them**, so the daily editing loop
(define once, select on the CLI, validate before compute) matches how the application is actually
run. The single highest-value move is **(C) separating selection from definition**: most of the
per-run file-editing friction is selections masquerading as config.
