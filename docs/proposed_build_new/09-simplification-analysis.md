# 09 · Simplification analysis (a strict critical review)

The preceding pages describe a capable but **heavy** build. This page is the adversarial pass:
it assumes the plan is over-engineered until each piece proves it earns its cost, judges every
major structure against the *actual* usecase, and recommends what to cut, downgrade, or keep —
with the trigger that would later justify reintroducing the complexity.

> This page may contradict earlier pages on purpose. Where it does, it says so and leaves the
> call to you. The other pages describe the *full* design; this one argues for the *lean default*
> and a path to grow into the full design only when a real need appears.

## How I'm judging (state the usecase, or the verdict is meaningless)

Every "is this too complex?" answer depends on the usecase. Assumptions, from the docs +
project memory — **correct me if any is wrong, because some verdicts flip:**

- **One small research team**, not multiple teams shipping components to each other.
- **One cluster, one container.** A single SIF runs the whole DAG; there is no scenario where
  `histomil-training` is released to an external consumer on a different cadence than
  `histomil-evaluation`.
- **A handful of dataset sources** (prostate; `sahlgrenska_2018`, `umea_2018`), not hundreds.
- **A40-class GPUs, <1B-param models, ample RAM** — bags fit in memory; OOM/chunking is not a
  live threat.
- **The scientific output is the point**: honest CV-vs-holdout numbers, reproducibility, and
  inspectable attention. Engineering flexibility is only valuable insofar as it protects those.

The cleaving question I apply throughout:

> **Does this complexity protect scientific validity / reproducibility, or does it buy
> engineering flexibility we may never exercise?** The first is worth real cost. The second must
> be earned by a concrete, named trigger — not "we might one day."

## TL;DR verdict

1. **The single biggest simplification is to drop the 7-package uv workspace for one package with
   enforced internal boundaries.** It is the load-bearing complexity; most of the surrounding
   ceremony (per-component CLIs, the report-toolkit-in-`shared` contortion, the CI matrix, deeper
   imports, the manually-policed "star") exists *only* to serve it. A single `histomil` package
   with module boundaries enforced by `import-linter` delivers the same separation and
   independent-evolution experience at a fraction of the cost. **This reverses the earlier
   workspace decision** — see [the conflict flag](#conflict-with-earlier-decisions).
2. **Downgrade the content-addressed per-row embedding cache to a per-`(scan, patch_config)` file
   cache** unless overlap/patch-size sweeps are a *frequent* activity. The row-level cache +
   delta-fill + Snakemake-freshness coupling is a documented footgun bought for a benefit the
   draft config (`overlap: 0.0`, one patch size) may never trigger.
3. **Defer augmentation-in-preprocessing.** It multiplies embedding storage and adds a cache-key
   axis to serve a benefit (augmented training data) that foundation-model embeddings are largely
   robust to. Prove uplift on one target before paying for it.
4. **Keep, without apology:** the cohort/role/split model + frozen `membership_hash`, OOF
   checkpoint routing, the schemas on leakage-critical tables, provenance, the raw/rigid/elastic
   split, and BEAM. These are the parts that protect the science. Cutting them would be the real
   mistake.

## The complexity ledger

| Complexity | True cost | Claimed benefit | Triggers in *this* usecase? | Verdict |
|---|---|---|---|---|
| **7-package uv workspace** | High: 7 `pyproject`s, workspace sources, star discipline policed by hand, CLI×7, toolkit forced into `shared`, CI matrix, deeper imports | Independent build / version / release / dep-isolation per stage | **No** — one team, one SIF, one DAG | **Simplify** → single package + `import-linter` |
| Per-component CLIs | Medium | follows packaging | No | Folds away with the workspace → one `histomil` CLI |
| Report toolkit pushed into `shared` | Medium (conceptual contortion in [07](07-reports.md#where-the-toolkit-lives-a-star-constraint-consequence)) | only exists to satisfy the star | n/a | **Cut** — disappears once packaging is one unit |
| **Per-row content-addressed cache + delta-fill** | High: freshness-vs-DAG coupling, canonical-order invariant, a known footgun ([04](04-snakemake-integration.md#embedding-cache-vs-the-dag)) | reuse embeddings across overlap/patch-size sweeps | **Rarely** — draft is one patch config, `overlap: 0.0` | **Downgrade** to per-`(scan, patch_config)` file cache; keep content-addressing only on the augmentation axis if kept |
| **Augmentation as own embedding sets** | High storage (×`n_variants`) + cache-key axis + config surface | augmented MIL training data | **Unproven** — FM embeddings are fairly aug-invariant | **Defer** until measured uplift |
| pandera **and** pydantic | Low–Med: two schema tools | executable dataframe + config contracts matching acceptance criteria | Yes — design is invariant-heavy | **Keep** (could collapse to pydantic-only if you prefer one tool) |
| 8+ per-stage config files | Medium: file sprawl | edit-one-stage ergonomics | Partially | **Keep**, but a single layered config with sections is defensible for one team |
| Full HPO infra (Optuna + promote + segregate) | Medium | systematic search | Later | **Keep but defer** (already Phase 7) |
| `shared` split into `io` / `formats` / `schemas` | Low | clarity | — | **Merge** acceptable; not worth a rule |
| Cohort/role/split + `membership_hash` | Medium | leakage-proof, comparable CV-vs-holdout | **Yes — core** | **Keep — non-negotiable** |
| OOF checkpoint routing | Medium | honest CV-test score | **Yes — core** | **Keep — non-negotiable** |
| raw/rigid/elastic variants | High (registration) | aligned heatmaps without touching metrics | Yes (design-core) | **Keep** |
| BEAM custom HDF5 | Medium | appendable per-biopsy ragged results + provenance | Yes | **Keep** |
| Seed sweep (fold × model seed) | Medium compute | spread, not a single lucky run | Yes — scientific | **Keep** |

---

## Deep dive 1 · The packaging is the complexity

**What it is.** Seven installable distributions in a uv workspace, each with its own `pyproject`,
console script, config dir, and Snakemake rules, all depending on `histomil-shared`, with a
hand-policed rule that no stage may import another.

**The honest cost.** It is not just "seven folders." It is:

- A *dependency-graph invariant policed by humans*. Nothing mechanically stops
  `histomil-training` from importing `histomil-evaluation`; the star holds only by vigilance.
- **Complexity that breeds complexity.** The clearest tell: `resolve_cohort` lives in
  preprocessing but emits an HTML report, so the entire report toolkit had to be exiled into
  `shared` behind a `[reports]` extra ([07](07-reports.md#where-the-toolkit-lives-a-star-constraint-consequence)),
  and `preprocessing` now drags plotly/jinja2. That contortion exists for **no reason the science
  cares about** — purely to keep the package graph acyclic. When a design forces you to relocate
  unrelated code to satisfy a structural rule, the structure is too expensive.
- Seven CLIs, a multi-package CI matrix, deeper import paths, and a real onboarding tax (a
  contributor must understand uv workspaces before changing one line).

**What it buys.** Genuinely: independent *distribution* and *dependency isolation*. If
`histomil-training` were pip-installed by an outside group, or shipped on its own release
cadence, or had a dependency that *conflicts* with evaluation's, the workspace is the right tool.

**Does the usecase trigger it?** No. One team, one SIF baked from one lock, one DAG. The
components are never installed apart in practice; the SIF carries all of them anyway
([02](02-environment-and-packages.md#containers) already admits "a SIF carries all components").
So we pay the full cost of separability and use ~none of it.

**Simpler alternative — same separation, far less cost:**

```text
src/histomil/
  shared/  wsi_transformation/  preprocessing/  training/  evaluation/  heatmaps/  reporting/
```

One `pyproject`, optional-dependency *extras* per stage (`histomil[train]`, `histomil[wsi]`), one
`histomil` CLI with subcommands, and **`import-linter`** contracts in CI to enforce exactly the
star ("no module imports a sibling stage; stages may import `shared`"). You keep: clear
boundaries, independent code evolution, the ability to test a stage in isolation, and a
mechanically-enforced dependency rule. You lose: independent versioned *releases* — which you
don't do.

**Trigger to adopt the workspace later:** a component gets an external consumer, a separate
release cadence, or a hard dependency conflict with another stage. Splitting a clean single
package (with boundaries already enforced) into a workspace later is mechanical; merging a
workspace back is not. **Start single, graduate on trigger.**

## Deep dive 2 · The embedding cache is solving a problem you may not have

**What it is.** A per-row, content-addressed embedding store keyed by `sha1(coords ∥ patch_size ∥
mpp ∥ model ∥ variant ∥ aug)`, with delta-fill on overlap changes, plus the machinery to keep
that row-level reuse consistent with Snakemake's file-level DAG and a *canonical row order*
invariant so attention still lines up with coordinates.

**The honest cost.** [04 · cache vs. DAG](04-snakemake-integration.md#embedding-cache-vs-the-dag)
spends a whole section on the failure mode; the canonical-order requirement is a latent
attention-misalignment bug if any reassembly path gets it wrong. This is real, sharp machinery.

**What it buys.** Reuse when you change *overlap* or *patch size* and the grids partially
overlap — you re-embed only the new positions.

**Does the usecase trigger it?** The draft `preprocessing.yaml` has **one** patch size and
**`overlap: 0.0`**. Embedding is a one-time cost per `(scan, variant, model)`; the foundation
model is the expensive part and you run it once. If you sweep patch configs rarely, a
**per-`(scan, patch_config)` file cache** — "this exact config already embedded? skip the rule" —
captures ~all the savings with none of the row-level hazards.

**Recommendation.** Default to file-level caching keyed by the full patch config. Keep
content-addressing *only* if/when you genuinely sweep overlap, and even then prefer the design's
own rejected-but-simpler fallback (embed once at the finest grid, decimate for coarser) over
per-row delta-fill. **Trigger:** a real experiment plan that varies overlap/patch size across
many runs on the same scans.

## Deep dive 3 · Augmentation may not earn its storage

**What it is.** Each augmentation is embedded through the foundation model and cached as its own
embedding set (`augmentation_id` in the key); training chooses whether to sample them.

**The honest cost.** Storage ×`n_variants` for every augmented scan, an extra cache-key axis, a
config block, and bag-store/fold logic to keep augmented bags in *train only*.

**What it buys.** More training signal — *if* augmentation helps MIL on top of a frozen
foundation encoder.

**The strict question.** Self-supervised histology foundation models are trained to be fairly
invariant to exactly these transforms (flips/rotations/stain jitter). Augmenting *pixels then
re-embedding* often yields embeddings close to the original — so the marginal value is unclear,
while the storage cost is concrete and large.

**Recommendation.** Build the hooks (the design's placement in preprocessing is correct *if* you
augment at all), but **ship the first version with augmentation off** and treat "does augmentation
improve the held-out metric on one target?" as an early experiment. Pay the storage only after a
measured win. **Trigger:** demonstrated uplift.

## Deep dive 4 · Schemas and configs — mostly keep, trim the edges

- **pandera + pydantic.** Two tools, but they cover different shapes (dataframes vs nested
  config), and the design is unusually invariant-heavy — leakage checks, membership, fold
  composition are *dataframe* contracts that map one-to-one onto acceptance criteria. This is
  complexity that protects validity. **Keep.** If one-tool simplicity is preferred, pydantic can
  validate rows too, at some ergonomic loss; not worth forcing.
- **Per-file configs.** The "edit one stage" ergonomic is real, but 8+ files plus two shared
  registries is a lot for one team. A single `config.yaml` with per-stage sections (the design's
  own listed alternative) is defensible and reduces "which file?" friction. **Mild simplify
  candidate**, low stakes either way.
- **`shared.io` vs `shared.formats` vs `shared.schemas`.** Three homes for "data shape" code. Fine
  to **merge** `io` + `formats`; don't make a doctrine of the split.

---

## What complexity is genuinely necessary (and why)

Being strict cuts both ways — these are worth their cost because they defend the *result*, and a
"simpler" pipeline that drops them would be cheaper and wrong:

- **Cohort / role / split + frozen `membership_hash`.** This is the crown jewel. It is what makes
  "CV-test score" and "holdout score" honest and comparable, and what makes leakage *auditable*.
  Every gram of this complexity buys scientific credibility. **Keep all of it.**
- **OOF checkpoint routing** (dev → out-of-fold checkpoint; holdout → ensemble). Removing it
  silently inflates metrics. Non-negotiable.
- **No fitted statistics in bundles; fit per-train-fold only.** Cheap to state, essential to
  honor; the bundle writer simply must have no field for fitted state.
- **Provenance everywhere** (git commit, config hash, membership hash). Reproducibility is the
  stated reason the project is a pipeline at all.
- **raw/rigid/elastic variants.** Heavy, but it is the whole point of decoupling heatmap visuals
  from metrics; the design's central principle rests on it.
- **BEAM.** A custom appendable HDF5 with ragged attention/outline arrays + provenance genuinely
  fits the per-biopsy result shape better than parquet-of-scalars. **Keep.**
- **Seed sweep.** Reporting a spread instead of one lucky run is core scientific hygiene.

The pattern: **complexity that protects validity stays; complexity that buys unused engineering
flexibility goes.**

## Two reference layouts

**Lean default (recommended start):**

```text
src/histomil/{shared, wsi_transformation, preprocessing, training, evaluation, heatmaps, reporting}/
one pyproject (extras per stage) · one `histomil` CLI · import-linter star · one config.yaml (sections)
file-level embedding cache · augmentation OFF · pandera+pydantic on leakage-critical artifacts
— full design retained for: cohorts/roles/splits, OOF routing, provenance, variants, BEAM, seed sweep
```

**Full design (graduate into it on triggers):** the uv workspace
([01](01-repository-layout.md)), per-row content-addressed cache, augmentation sets, per-file
configs — adopt each *only* when its named trigger fires:

| Adopt… | …when |
|---|---|
| uv workspace | a component gets an external consumer / separate release / conflicting deps |
| per-row cache + delta-fill | overlap/patch-size sweeps become frequent on shared scans |
| augmentation sets | augmentation shows a measured held-out uplift |
| per-file config split | the single config genuinely becomes unwieldy across many concurrent stage runs |

Going lean→full is mechanical (a clean single package with enforced boundaries *splits* cleanly).
Going full→lean is not. That asymmetry is the whole argument for starting lean.

## Conflict with earlier decisions

This page **reverses the uv-workspace choice** made earlier in planning. That choice optimized for
"updated independently"; this review argues the *code-separation* half of that goal is fully met
by a single package with `import-linter`, while the *distribution-independence* half is unused in
the stated usecase — so the workspace is cost without payoff **today**, and is a clean later
upgrade **if** a trigger appears. If the usecase assumptions are wrong — e.g. separate teams own
separate stages, or stages will be released/installed independently — then the workspace verdict
flips back and the earlier decision stands. That is the one fact worth confirming before acting.

## Recommended actions now

1. **Decide the packaging** against the usecase assumptions above — it's the highest-leverage
   call and everything else is downstream of it. Default recommendation: single package +
   `import-linter`.
2. **Default the embedding cache to file-level**; gate the per-row cache behind a real
   overlap-sweep need.
3. **Ship augmentation off**; make it experiment #1, not infrastructure #1.
4. **Keep the validity-protecting core exactly as designed** — do not let "simplification"
   anywhere near cohorts/roles/splits, OOF routing, provenance, or BEAM.
