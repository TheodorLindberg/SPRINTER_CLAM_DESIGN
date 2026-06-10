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
| `register` | patient | normalized scans → `processed/{dataset}/{patient}/registered/{scan}.{variant}.ome.tiff` + `transform.json` |
| `detect_outline` | scan × variant | image → `…/outlines/{scan}__{variant}.geojson` (+ polygon array) |
| `cross_stain_intersection` | scan | elastic outlines → `…/outlines/{scan}__intersection.geojson` |
| `biopsy_axis` | scan | mask/outline → `…/axis/{scan}.json` (PCA axis + quartile cuts) |

### Stage 3 · Dataset Preprocessing — `preprocessing.yaml`

| Rule | Per | Inputs → Outputs |
|---|---|---|
| `derive_labels` | dataset | raw labels → `processed/{dataset}/labels_derived.csv` |
| `patch_coords` | scan × variant × patch_config | outline + axis → `…/coords/{patch_config}/{scan}__{variant}.h5` |
| `embed` | scan × variant × patch_config × embedding_model × aug | coords + WSI → `…/embeddings/{embedding_model}/{aug}/{scan}__{variant}__{patch_config}.h5` (content-addressed cache) |
| `assemble_bundle` | bundle | embeddings + derived labels + cohort → `bundles/{bundle_id}/` (manifest, labels, symlinks, metadata) |

### Stage 4 · Model Training — `model_experiment.yaml`, `hpo.yaml`, `seeds.yaml`

| Rule | Per | Inputs → Outputs |
|---|---|---|
| `generate_folds` | seed_set × fold_seed | cohort dev patients + seeds → `results/folds/{seed_set}/{fold_seed}.csv` |
| `train_run` | run_id × fold_seed × model_seed | bundle + folds → `results/experiments/{exp}/sweep/{run_id}/` (checkpoints, metrics, `run.json`) |
| `aggregate_runs` | — | all `run.json` → `results/runs.parquet` |
| `hpo` | hpo name | bundle + search space → `results/experiments/{name}/hpo/` (Optuna; segregated, top-N) |

### Stage 5 · Evaluation — `evaluation.yaml`

| Rule | Per | Inputs → Outputs |
|---|---|---|
| `infer` | scan | model + bundle (subset) → `results/evaluation/{eval}/per_scan/{scan}.h5` |
| `aggregate_beam` | biopsy × model | per-scan H5 → `…/beam/{biopsy}__{model}.beam.h5` |

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
    M[scan manifest<br/>Stage 1, external] --> REG[register]
    REG --> OUT[detect_outline]
    OUT --> AX[biopsy_axis]
    OUT --> XS[cross_stain_intersection]

    AX --> PC[patch_coords]
    OUT --> PC
    PC --> EMB[embed]
    LB[derive_labels] --> BUN
    EMB --> BUN[assemble_bundle]

    BUN --> TR[train_run]
    FG[generate_folds] --> TR
    BUN --> HPO[hpo]
    HPO -. promote top-N .-> TR
    TR --> AGG[aggregate_runs]

    BUN --> INF[infer]
    TR --> INF
    INF --> BEAM[aggregate_beam]

    BEAM --> HM[heatmap]
    REG --> HM
    OUT --> HM

    AGG --> REP[report]
    BEAM --> REP
```

## Named targets

Each stage exposes a target so it can run alone; `all` runs the chain.

```text
transform · outlines · coords · embeddings · bundles
folds · train · hpo · evaluate · heatmaps · reports · all
```

## Dynamic fan-out (checkpoints)

Some rule sets are unknown until inputs are read, so they sit behind Snakemake **checkpoints**:

- **Cohort resolution** — which `(patient, biopsy, stain)` bags actually exist (and their roles) is read from the frozen cohort membership before `assemble_bundle` / `train_run` targets resolve.
- **Model-experiment expansion** — the `runs` list × `fold_seeds` × `model_seeds` expands into concrete `train_run` jobs.
- **Evaluation set** — the biopsies present in a bundle determine the `aggregate_beam` jobs.

## Configuration → rules

| Config | Drives |
|---|---|
| `base.yaml` | roots + registries for **every** rule |
| `wsi_transformation.yaml` | `register`, `detect_outline`, `biopsy_axis`, `cross_stain_intersection` |
| `preprocessing.yaml` | `derive_labels`, `patch_coords`, `embed`, `assemble_bundle` |
| `seeds.yaml` | `generate_folds` |
| `model_experiment.yaml` | `train_run`, `aggregate_runs` |
| `hpo.yaml` | `hpo` |
| `evaluation.yaml` | `infer`, `aggregate_beam` |
| `heatmaps.yaml` | `heatmap` |
| `reports.yaml` | `report` |
