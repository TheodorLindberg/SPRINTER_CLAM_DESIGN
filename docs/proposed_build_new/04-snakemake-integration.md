# 04 · Snakemake integration

Builds the orchestration described in [`impl/workflow.md`](../impl/workflow.md):
stages 2–6 as independently runnable named targets, one top-level chain, config layering,
checkpoints for runtime-discovered fan-out, and SLURM dispatch. Stage 1 stays outside.

## Structure: one thin `workflow/`, one `.smk` per stage

The workflow holds no algorithm — just rules that marshal wildcards → paths → a `histomil`
subcommand. One `.smk` per stage keeps it readable:

```text
workflow/
├─ Snakefile               # includes rules/*.smk; defines `rule all`
├─ rules/
│  ├─ common.smk           # configfile loading, wildcard_constraints, Paths() helper, resources
│  └─ transform.smk  preprocess.smk  train.smk  evaluate.smk  heatmap.smk  reports.smk
└─ profiles/slurm/config.yaml
```

```python
# workflow/Snakefile
include: "rules/common.smk"
include: "rules/transform.smk"
include: "rules/preprocess.smk"
include: "rules/train.smk"
include: "rules/evaluate.smk"
include: "rules/heatmap.smk"
include: "rules/reports.smk"
```

`common.smk` is the only place that touches Snakemake's `config`:

```python
# rules/common.smk
configfile: "config/base.yaml"          # always layered first
# the rest added on the CLI: --configfile config/base.yaml config/pipeline.yaml
from histomil.shared.config import BaseConfig
from histomil.shared.paths import Paths
BASE  = BaseConfig.model_validate(config)
PATHS = Paths(BASE.roots)

wildcard_constraints:
    variant   = "raw|rigid|elastic",
    stain     = "HE|Ki67|PSA",
    fold_seed = r"\d+", model_seed = r"\d+",
```

## Rules call the package, never inline logic

Every rule body is a one-line `histomil <stage> <cmd>` call (or a tiny `run:` that calls a stage
function). All algorithm lives in the package, so it is unit-testable without Snakemake.

```python
# rules/preprocess.smk  (representative)
rule embed:
    input:
        coords = lambda wc: PATHS.coords(wc.dataset, wc.patient, wc.scan, wc.variant, wc.patch_config),
        wsi    = lambda wc: scan_path(wc),                 # from the scan manifest, not the tree
    output:
        PATHS.embeddings("{scan}", "{variant}", "{patch_config}", "{embedding_model}")
    resources: gpu=1, mem_mb=32000, runtime=120
    group: "embed_{dataset}_{patient}"                     # job grouping — see below
    shell:
        "histomil preprocess embed --config {CONFIGS} "
        "--coords {input.coords} --wsi {input.wsi} --out {output} "
        "--model {wildcards.embedding_model}"
```

The mapping of which config drives which rule is fixed in
[`impl/workflow.md` → Configuration → rules](../impl/workflow.md#configuration-rules) and
reproduced by `common.smk` passing the right `--configfile` set per stage.

## Wildcards (the DAG axes)

Taken verbatim from the [workflow doc](../impl/workflow.md#wildcards-the-dag-axes):
`dataset, patient, biopsy, scan, stain, variant, patch_config, embedding_model, cohort,
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

## File-level embedding cache

The embedding file path encodes the full config (`{scan}__{variant}__{patch_config}` under the
model), so the cache **is** the DAG: Snakemake skips a configuration whose file already exists and
rebuilds only the ones that changed. `embed` is an ordinary rule writing rows once in coordinate
order — the same order training/eval feed the model and BEAM writes as `patches/coords`
([evaluation invariant](../spec/evaluation.md#invariants)) — so attention and coordinates line up
by construction, with no separate store to keep consistent.

## Job grouping

The `scan × variant × patch_config × embedding_model` matrix is many short jobs. Per the
workflow doc, `group:` directives pack `patch_coords`/`embed` tasks into one SLURM allocation
(grouped by patient here), processing several scans per warm-GPU worker. The aim is a few
well-sized jobs, not thousands of seconds-long ones.

## SLURM profile & launch

`scripts/run.sh` submits the Snakemake **controller**; the cluster-generic / SLURM-executor
profile dispatches one worker per rule inside the SIF (`container:` directives share the
environment). Heavy rules (`embed`, `train_run`, `infer`) request `gpu=1`.

```bash
sbatch scripts/run.sh embeddings configfile=config/pipeline.yaml
sbatch scripts/run.sh all
```

`profiles/slurm/config.yaml` sets default `resources`, retries, and the executor; per-rule
`resources:` override it. This wiring is ported largely as-is from the old pipeline's
`util/slurm/*` (old-pipeline review §3) — it is hard-won infra.
