# Appendix — Design Decisions

The reasoning behind the spec — the "why" — kept out of the stage docs so those stay focused on the design itself.

## How `dataset_id` is meant to work

`dataset_id` **names the source of a dataset, and may carry a version** — e.g. `sahlgrenska_2018`, `umea_2018`, `karolinska_v2`. It is *not* just a version string.

- A patient is globally `(dataset_id, patient_id)`, so the same `dataset_id` is what lets a [cohort](02-data-model.md#cohorts-roles-and-splits) pool patients from several sources unambiguously.
- Never `latest` — the id must pin a specific source/version so provenance is stable.

---

## Design decisions

Each entry: **what** we decided and **why**.

### Data & contracts

- **The manifest is the contract, not the disk layout.** Ingestion emits a scan manifest (IDs → WSI path); the pipeline never parses folder/flat layout. *Why:* ingestion is user-written and varied; decoupling keeps it flexible.
- **Metadata rides on the manifest, at any entity level, and is forwarded** through preprocessing into the bundle and BEAM. *Why:* no separate metadata store; available everywhere downstream without re-joining the source.
- **`dataset_id` = source (+ optional version).** See above.

### Cohorts, roles, splits

- **Collapsed "patient set" into "cohort".** A cohort is the named patient group; `development`/`holdout` are per-patient **roles**, not sub-cohorts. *Why:* one fewer term; "cohort" matches clinical usage; removes the overloaded "test".
- **`train`/`val`/`test` are fold roles only; the locked patients are `holdout`.** *Why:* a "CV test score" and a "holdout score" must never be confused.
- **Splits are over the cohort's development patients** (patient-level), with frozen, hashed membership. *Why:* every stain/embedding bundle inherits the *same* fold split; membership changes are detected.
- **No fitted statistics in bundles.** Normalization/thresholds are computed at training time from the training fold only. *Why:* makes holdout leakage-free by construction.

### Bundles

- **A bundle is a prepared cohort** for one (stain · embedding · patching · source variant); every patient present, each bag tagged by role. One cohort → many bundles. *Why:* preprocessing runs independently and produces a frozen, validated, reusable artifact; stages pick a `subset` (`development`/`holdout`/`all`) rather than juggling a list.

### Images & embeddings

- **Source variants `raw`/`rigid`/`elastic`.** Training and all metrics use `raw`; registration affects **heatmaps only**. *Why:* elastic distortion changes visuals, never the reported numbers.
- **File-level embedding cache** — one HDF5 per scan × source variant × embedding model × patch config, tracked directly by the workflow (the path is the key). *Why:* reuse is visible to the Snakemake DAG with no separate store; an already-embedded config is skipped and reused across every cohort that needs it, and a changed config writes a new file.

### Formats

- **BEAM** — the project's per-biopsy, per-sweep HDF5 evaluation format; appendable; outline stored as polygon arrays; every contributing model in the sweep keeps its own prediction/attention/stats under `models/{run_id}/`. *Why:* one self-describing, extensible result file per biopsy, not fragmented across every model in a sweep.
- **Binary is the source of truth; GeoJSON is the view** (TissUUmaps). *Why:* performance + a clean separation from visualization.

### Experiments, runs, reports

- **Runs are generated, not configured.** A `model_experiment` is shared defaults + explicit runs (each fanning out over the seed sweep); HPO is an optional `hpo` block in the same experiment file that fans out into trials. *Why:* config count stays O(experiments), not O(models) — HPO would otherwise mean hundreds of files.
- **`base.yaml` holds roots.** *Why:* change an output path once, not in every stage config.
- **HPO is segregated** (own index, top-N checkpoints kept); workflow is HPO → promote best → seed sweep. *Why:* sweep models are the keepers; HPO models are exploratory and rarely revisited.
- **Reports are a regenerable view** over manifests / `runs.parquet` / BEAM — static HTML, Plotly, standalone CSS, faceted by tags, with two-level data export. *Why:* reproducible, archival, and the real backing artifacts stay the source of truth.
