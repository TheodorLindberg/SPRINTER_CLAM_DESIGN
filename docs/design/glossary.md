# Glossary

Definitions of the terms and abbreviations used across the spec. Where a term has its own treatment, the linked doc is authoritative.

## Data, cohorts & splits

| Term | Meaning |
|---|---|
| **Dataset** | The patients, biopsies, scans, and labels from one **source**. `dataset_id` names the source and may carry a version (e.g. `sahlgrenska_2018`). |
| **Patient** | A biological individual, identified globally by `(dataset_id, patient_id)`. `patient_id` is an **arbitrary index within the dataset** (e.g. `p0001`) — deliberately **not** a real patient / PAD / scan number, so the data stays de-identified. |
| **Cohort** | A named, possibly multi-dataset **group of patients**. Splits and bundles derive from it. See [Data Model](02-data-model.md#cohorts-roles-and-splits). |
| **Role** | A patient's place in a cohort: `development` (cross-validation) or `holdout` (locked test). |
| **Development** | The role/patients used for model development; all cross-validation happens here. |
| **Holdout** | The role/patients locked away from development and scored once at the end ("lockbox"). |
| **Split** | Fold-level assignment among development patients: `train` / `val` / `test`. |
| **Fold** | One cross-validation partition; `train`/`val`/`test` are *fold roles*, never the holdout. |
| **Seed set** | A named split configuration ([`seeds.yaml`](../configs/seeds.md)) over a cohort: fold count, fold seeds, model seeds. |
| **Fold seed / model seed** | Seeds varied independently in the [seed sweep](06-model-training.md#seed-sweep): the fold seed sets the partition, the model seed sets weight initialization. |
| **Biopsy** | A tissue sample from one patient; the unit labels and bags are keyed on. |
| **Label** | A target on a biopsy — `name`, `type`, `value`. Optional. **Derived labels** are computed from raw scores in preprocessing. |
| **Score quartile** | One of the 4 per-biopsy IHC scores (Ki67/PSA). Spatially unlocated, so **averaged** to a biopsy value; carried only as metadata. |
| **Geometric quartile** | A spatial 4-way split of the tissue along the [biopsy axis](#images-patches--embeddings) (PCA line). Used for heatmap regions, **not** mapped to score quartiles. |

## Images, patches & embeddings

| Term | Meaning |
|---|---|
| **Scan** | A digitized WSI from one biopsy and one stain. |
| **Stain** | The staining method on a scan (H&E, IHC stains). |
| **Source variant** | A scan's form: `raw` (training/metrics), `rigid`, or `elastic` (registered, for heatmaps). See [Data Model](02-data-model.md#source-variants). |
| **Registration** | Aligning stains; `rigid` (rotation/translation) or `elastic` (warps to H&E). |
| **Tissue outline** | Polygon array of tissue extent per variant; the **cross-stain intersection** is tissue present in every stain. See [Outlines](../formats/outlines.md). |
| **Biopsy axis** | The PCA longitudinal line of a biopsy (in the raw/training frame); defines the geometric quartiles. See [WSI Transformation spec](../spec/wsi-transformation.md#biopsy-axis-pca-line). |
| **Patch** | A crop from a scan at a configured size and resolution. |
| **Embedding** | A feature vector from a patch, produced by an embedding model. |
| **Content-addressed cache** | Embedding store keyed by coords + size + resolution + model + variant + augmentation; only misses are embedded. |
| **Augmentation** | Image-space transforms applied **before** embedding (runs the foundation model), cached as separate sets. |

## Training & evaluation

| Term | Meaning |
|---|---|
| **Bag** | The set of patch embeddings used as one model input instance. |
| **Bundle** | A **prepared cohort** for one (stain · embedding · variant · patch config); every patient present, tagged by role. One cohort → many bundles. |
| **Subset** | Which role a stage uses: `development`, `holdout`, or `all` (the union, for a final retrain). |
| **Model experiment** | Shared `defaults` + explicit **runs** that fan out over the seed sweep ([`model_experiment.yaml`](../configs/model_experiment.md)). |
| **Run** | One configuration within a model experiment (`run_id`), itself fanning out over the seed sweep. |
| **Seed sweep** | Training across all fold-seed × model-seed combinations. |
| **HPO** | Hyperparameter optimization — a separate search ([`hpo.yaml`](../configs/hpo.md)); outputs segregated, top-N kept. |
| **Checkpoint** | A saved trained model (per fold). |
| **Attention** | Per-patch importance from the MIL model: `raw`, `sigmoid`, `rank`. |
| **Heatmap** | Attention/prediction rendered over a chosen source variant. |

## Artifacts & formats

| Term | Meaning |
|---|---|
| **Manifest** | The IDs → paths table that is the pipeline contract; carries metadata columns at any level. |
| **BEAM** | The per-biopsy, per-model HDF5 evaluation result format. See [BEAM](../formats/beam.md). |
| **`runs.parquet`** | The master, tagged index of every run — the report backbone and export table. |
| **Provenance / membership hash** | Recorded inputs (config, git commit, seeds) and the cohort's frozen-membership hash used for stale-split detection. |

## Abbreviations

| Abbrev. | Expansion |
|---|---|
| **WSI** | Whole-slide image |
| **MIL** | Multiple-instance learning |
| **H&E / HE** | Hematoxylin & eosin (stain) |
| **IHC** | Immunohistochemistry |
| **Ki67 / PSA** | IHC markers used as stains (PSA also a serum-value label) |
| **mpp** | Microns per pixel (resolution) |
| **CV** | Cross-validation |
| **HPO** | Hyperparameter optimization |
| **TPE** | Tree-structured Parzen estimator (a Bayesian optimizer) |
| **HED** | Hematoxylin–eosin–DAB colour space (stain jitter) |
| **CLAM** | Clustering-constrained attention MIL (a model family) |
| **VALIS** | WSI registration toolkit (also supplies tissue masks) |
| **TissUUmaps** | Interactive WSI viewer (renders GeoJSON layers) |
| **BEAM** | Biopsy Evaluation & Attention Map (this project's result format) |
| **HDF5** | Hierarchical Data Format v5 (binary container) |
| **GeoJSON** | JSON geometry format (TissUUmaps viewing) |
| **MAE / R² / AUROC / F1** | Regression & classification metrics |
