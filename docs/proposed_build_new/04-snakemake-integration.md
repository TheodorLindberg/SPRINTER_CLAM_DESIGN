# 04 · Snakemake integration

Builds the orchestration described in [`impl/workflow.md`](../impl/workflow.md):
stages 2–6 as independently runnable named targets, one top-level chain, config layering,
checkpoints for runtime-discovered fan-out, and SLURM dispatch. Stage 1 stays outside.

## Structure: rules live with their component

Each component owns its `workflow/rules.smk` (so the rules version with the code that backs
them). The thin top-level `workflow/` only wires them together:

```text
workflow/
├─ Snakefile               # includes each component's rules.smk; defines `rule all`
├─ rules/common.smk        # configfile loading, wildcard_constraints, Paths() helper, resources
└─ profiles/slurm/config.yaml

components/<name>/workflow/rules.smk   # that component's rules (register, embed, train_run, …)
```

```python
# workflow/Snakefile
include: "rules/common.smk"
include: "../components/wsi_transformation/workflow/rules.smk"
include: "../components/preprocessing/workflow/rules.smk"
include: "../components/training/workflow/rules.smk"
include: "../components/evaluation/workflow/rules.smk"
include: "../components/heatmaps/workflow/rules.smk"
include: "../components/reporting/workflow/rules.smk"
```

A component can also run **standalone** — `snakemake -s components/training/workflow/rules.smk`
after `include`-ing `common.smk` — so a stage is developed and run without the whole chain.

`common.smk` is the only place that touches Snakemake's `config`:

```python
# rules/common.smk
configfile: "config/base.yaml"          # always layered first
# stage configs added on the CLI: --configfile config/base.yaml config/<stage>.yaml
from histomil.shared.config import BaseConfig
from histomil.shared.paths import Paths
BASE  = BaseConfig.model_validate(config)
PATHS = Paths(BASE.roots)

wildcard_constraints:
    variant   = "raw|rigid|elastic",
    stain     = "HE|Ki67|PSA",
    aug       = "none|[a-z0-9_]+",
    fold_seed = r"\d+", model_seed = r"\d+",
```

## Rules call the package, never inline logic

Every rule body is a one-line call to that component's console script (or a tiny `run:` that
calls a stage function). All algorithm lives in the component package so it is unit-testable
without Snakemake.

```python
# components/preprocessing/workflow/rules.smk  (representative)
rule embed:
    input:
        coords = lambda wc: PATHS.coords(wc.dataset, wc.patient, wc.scan, wc.variant, wc.patch_config),
        wsi    = lambda wc: scan_path(wc),                 # from the scan manifest, not the tree
    output:
        PATHS.embeddings("{scan}", "{variant}", "{patch_config}", "{embedding_model}", "{aug}")
    resources: gpu=1, mem_mb=32000, runtime=120
    group: "embed_{dataset}_{patient}"                     # job grouping — see below
    shell:
        "histomil-preprocess embed --config {CONFIGS} "
        "--coords {input.coords} --wsi {input.wsi} --out {output} "
        "--model {wildcards.embedding_model} --aug {wildcards.aug}"
```

Because the rule shells out to the component's console script, the workflow needs the components
*installed* (the SIF has them all) but never *imports* them — so the orchestration layer carries
no stage code and a component's rules can't reach into a sibling.

The mapping of which config drives which rule is fixed in
[`impl/workflow.md` → Configuration → rules](../impl/workflow.md#configuration-rules) and
reproduced by `common.smk` passing the right `--configfile` set per stage.

## Wildcards (the DAG axes)

Taken verbatim from the [workflow doc](../impl/workflow.md#wildcards-the-dag-axes):
`dataset, patient, biopsy, scan, stain, variant, patch_config, embedding_model, aug, cohort,
bundle_id, seed_set, fold_seed, model_seed, model_experiment, run_id`. `scan = {biopsy}__{stain}`
is built by `shared.ids`, so the wildcard value and the id agree.

## Named targets

Each stage exposes a target; `all` chains them. These become `rule` aliases collecting the
stage's terminal outputs:

```text
register · cohort · coords · embeddings · bundles
folds · train · hpo · evaluate · heatmaps · reports · all
```

`cohort` is runnable on its own **before** heavy compute (it only needs the manifest + labels) —
the design's "sanity-check a cohort first" requirement.

## Checkpoints (runtime-discovered fan-out)

Three rule sets can't be enumerated until inputs are read, so they sit behind Snakemake
`checkpoint`s, exactly as the workflow doc specifies:

| Checkpoint | Freezes | Gates |
|---|---|---|
| `resolve_cohort` | which `(patient, biopsy, stain)` exist + roles + `membership_hash` | `assemble_bundle`, `generate_folds` |
| model-experiment expansion | `runs × fold_seeds × model_seeds` → concrete `train_run` jobs | `aggregate_runs` |
| evaluation set | biopsies present in a bundle → `aggregate_beam` jobs | `heatmap`, `report` |

Pattern (ported from the old pipeline's `discover_quartiles`, per the old-pipeline review
§3): the checkpoint writes a manifest CSV; a downstream
`input:` function reads it via `checkpoints.<name>.get(...).output` and expands the job list.

## Embedding cache vs. the DAG

The subtle bit from [`impl/workflow.md`](../impl/workflow.md#embedding-cache-vs-the-dag):
Snakemake judges the per-scan embedding file by existence/mtime, not by which coordinates it
holds. The build resolves it by **tying `embed`'s freshness to the coords file** (which encodes
`patch_config`): changing size/overlap rewrites coords → `embed` reruns, and inside,
`preprocess.embed` + `preprocess.cache` load the existing cache, embed only the new `(x,y)`
delta, copy reused rows, and rewrite in canonical order. So the rerun is cheap and correct
rather than all-or-nothing — and the canonical row order is the one training/eval feed the model
and BEAM writes ([evaluation invariant](../spec/evaluation.md#invariants)).

## Job grouping

The `scan × variant × patch_config × embedding_model × aug` matrix is many short jobs. Per the
workflow doc, `group:` directives pack `patch_coords`/`embed` tasks into one SLURM allocation
(grouped by patient here), processing several scans per warm-GPU worker. The aim is a few
well-sized jobs, not thousands of seconds-long ones.

## SLURM profile & launch

`scripts/run.sh` submits the Snakemake **controller**; the cluster-generic / SLURM-executor
profile dispatches one worker per rule inside the SIF (`container:` directives share the
environment). Heavy rules (`embed`, `train_run`, `infer`) request `gpu=1`.

```bash
sbatch scripts/run.sh embeddings configfile=config/preprocessing.yaml
sbatch scripts/run.sh all
```

`profiles/slurm/config.yaml` sets default `resources`, retries, and the executor; per-rule
`resources:` override it. This wiring is ported largely as-is from the old pipeline's
`util/slurm/*` (old-pipeline review §3) — it is hard-won infra.
