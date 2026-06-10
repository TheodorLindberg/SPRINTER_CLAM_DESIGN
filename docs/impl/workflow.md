# Impl · Snakemake workflow

How the stages become Snakemake rules — the rules, their wildcards, dependencies, and the config that drives each. This is the concrete map of *what gets built*.

## Orchestration model

- **Stage 1 (Ingestion) is outside Snakemake** — a user bridge writes the normalized scans + [scan manifest](../design/03-data-ingestion.md#scan-manifest-the-contract). The DAG starts from that manifest.
- **Stages 2–6 are Snakemake.** Each stage is independently runnable via a **named target**; one top-level workflow chains them.
- **Config layering:** [`base.yaml`](../configs/base.md) (roots + registries) is always loaded; the stage's config supplies the rest; the [`cohorts`](../configs/cohorts.md) and [`seeds`](../configs/seeds.md) registries resolve membership and splits. `--configfile base.yaml <stage>.yaml`.
- **Execution:** rules carry `resources:` (gpu/mem/runtime); a cluster-generic SLURM profile dispatches workers; heavy rules (`embed`, `train_run`) run on GPU.

## Wildcards (the DAG axes)

| Wildcard | Meaning |
|---|---|
| `dataset`, `patient`, `biopsy`, `scan` | entity ids (`scan` = biopsy × stain) |
| `stain` | `HE` / `Ki67` / `PSA` |
| `variant` | `raw` / `rigid` / `elastic` |
| `patch_config` | patch size · resolution · overlap |
| `embedding_model` | embedder id |
| `aug` | augmentation id (`none` = unaugmented) |
| `cohort`, `bundle_id` | named cohort; prepared-cohort id |
| `seed_set`, `fold_seed`, `model_seed` | split set + the two swept seeds |
| `model_experiment`, `run_id` | umbrella + one run |

## Rules by stage

### Stage 2 · WSI Transformation — `wsi_transformation.yaml`

| Rule | Per | Inputs → Outputs |
|---|---|---|
| `register` | patient | normalized scans → registered OME-TIFFs + `transform.json` + **outlines** `…__{variant}__{tissue_method}.geojson` (VALIS or hsv_otsu) + **cross-stain intersection** + **QC PNG** (outline-on-tissue, both methods when comparing) |
| `biopsy_axis` | scan | outline → `…/axis/{scan}.json` (PCA axis + quartile cuts) |

`register` does registration **and** outlines in one rule — VALIS already segments tissue and holds the transforms, so a separate outline rule would re-derive both. `biopsy_axis` stays separate (pure geometry on the outline); it can be inlined into `register` if preferred.

### Stage 3 · Dataset Preprocessing — `preprocessing.yaml`

| Rule | Per | Inputs → Outputs |
|---|---|---|
| `resolve_cohort` | cohort | `cohorts.yaml` entry + member manifests + derived labels → `processed/cohorts/{cohort}/membership.csv` (+ hash) + **validation** + `results/reports/cohorts/{cohort}.html` |
| `derive_labels` | dataset | raw labels → `processed/{dataset}/labels_derived.csv` |
| `patch_coords` | scan × variant × patch_config | outline + axis → `…/coords/{patch_config}/{scan}__{variant}.h5` |
| `embed` | scan × variant × patch_config × embedding_model × aug | coords + WSI → `…/embeddings/{embedding_model}/{aug}/{scan}__{variant}__{patch_config}.h5` (content-addressed cache) |
| `assemble_bundle` | bundle | embeddings + derived labels + **resolved cohort** → `bundles/{bundle_id}/` (manifest, labels, symlinks, metadata) |

`resolve_cohort` is the first thing preprocessing does: it freezes membership (so the hash can detect drift), validates it, and emits the cohort report — runnable on its own (`cohort` target) before any heavy compute.

### Stage 4 · Model Training — `model_experiment.yaml`, `hpo.yaml`, `seeds.yaml`

| Rule | Per | Inputs → Outputs |
|---|---|---|
| `generate_folds` | seed_set × target × fold_seed | cohort dev patients + seeds → `results/folds/{seed_set}/{target}/{fold_seed}.csv` (target in the key only when stratified) |
| `train_run` | run_id × fold_seed × model_seed | bundle + folds → `results/experiments/{exp}/sweep/{run_id}/` (checkpoints, metrics, `run.json`) |
| `aggregate_runs` | — | all `run.json` → `results/runs.parquet` |
| `hpo` | hpo name | bundle + search space → `results/experiments/{name}/hpo/` (Optuna; segregated, top-N) |

### Stage 5 · Evaluation — `evaluation.yaml`

| Rule | Per | Inputs → Outputs |
|---|---|---|
| `infer` | bag (per patient) | run + bundle (subset) → per-bag predictions/attention, via [out-of-fold / ensemble checkpoint routing](../spec/evaluation.md#checkpoint-routing-the-crux) |
| `aggregate_beam` | biopsy × run | per-bag results → `…/beam/{biopsy}__{run_id}.beam.h5` |

### Stage 6 · Heatmaps — `heatmaps.yaml`

| Rule | Per | Inputs → Outputs |
|---|---|---|
| `heatmap` | biopsy × variant | BEAM + WSI(variant) + transform + outline → `results/heatmaps/{biopsy}__{stain}.png` + `.geojson` |

### Reports — `reports.yaml`

| Rule | Per | Inputs → Outputs |
|---|---|---|
| `report` | — | `runs.parquet` + manifests + BEAM → `results/reports/` (HTML) |

## Dependency DAG

```mermaid
flowchart TD
    M[scan manifest<br/>Stage 1, external] --> REG[register<br/>+ outlines + QC PNG]
    REG --> AX[biopsy_axis]

    AX --> PC[patch_coords]
    REG --> PC
    PC --> EMB[embed]
    M --> RC[resolve_cohort<br/>freeze · validate · report]
    LB[derive_labels] --> RC
    RC --> BUN
    LB --> BUN
    EMB --> BUN[assemble_bundle]

    BUN --> TR[train_run]
    RC --> FG[generate_folds]
    FG --> TR
    BUN --> HPO[hpo]
    HPO -. promote top-N .-> TR
    TR --> AGG[aggregate_runs]

    BUN --> INF[infer]
    TR --> INF
    INF --> BEAM[aggregate_beam]

    BEAM --> HM[heatmap]
    REG --> HM

    AGG --> REP[report]
    BEAM --> REP
```

## Named targets

Each stage exposes a target so it can run alone; `all` runs the chain.

```text
register · cohort · coords · embeddings · bundles
folds · train · hpo · evaluate · heatmaps · reports · all
```

## Dynamic fan-out (checkpoints)

Some rule sets are unknown until inputs are read, so they sit behind Snakemake **checkpoints**:

- **Cohort resolution** — `resolve_cohort` freezes which `(patient, biopsy, stain)` exist and their roles; the resulting membership gates `assemble_bundle` and `generate_folds`.
- **Model-experiment expansion** — the `runs` list × `fold_seeds` × `model_seeds` expands into concrete `train_run` jobs.
- **Evaluation set** — the biopsies present in a bundle determine the `aggregate_beam` jobs.

## Configuration → rules

| Config | Drives |
|---|---|
| `base.yaml` | roots + registries for **every** rule |
| `cohorts.yaml` | `resolve_cohort` (validate + freeze + report) |
| `wsi_transformation.yaml` | `register` (registration + outlines + QC PNG), `biopsy_axis` |
| `preprocessing.yaml` | `derive_labels`, `patch_coords`, `embed`, `assemble_bundle` |
| `seeds.yaml` | `generate_folds` |
| `model_experiment.yaml` | `train_run`, `aggregate_runs` |
| `hpo.yaml` | `hpo` |
| `evaluation.yaml` | `infer`, `aggregate_beam` |
| `heatmaps.yaml` | `heatmap` |
| `reports.yaml` | `report` |
