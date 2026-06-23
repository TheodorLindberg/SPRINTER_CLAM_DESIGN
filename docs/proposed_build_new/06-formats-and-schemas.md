# 06 · Formats & schemas

Every artifact in the pipeline as a concrete, executable contract, mapped to its format spec.
**All definitions live in `histomil.shared`** — that is what lets the producing and consuming
stages stay independent while reading/writing against one contract. Hierarchical
configs/manifests get a `pydantic` model; tabular invariants get an explicit assertion function in
`histomil.shared.checks`; binary artifacts get a typed reader/writer in `histomil.shared.io`
(generic) or `histomil.shared.formats` (named cross-stage formats). The rule: a format is defined
**once**, and both the producer and the test suite import that one definition.

| Artifact | Format | Definition (`histomil.shared.…`) | Spec |
|---|---|---|---|
| Scan manifest | YAML/JSON | `manifest` (pydantic) | [data-ingestion](../spec/data-ingestion.md) |
| Raw label tables | CSV/Parquet | `checks.labels` + `io.tables` | [data-ingestion](../spec/data-ingestion.md#label-tables-optional) |
| Derived labels (long) | CSV | `checks.labels` | [preprocessing](../spec/preprocessing.md#bundle-a-prepared-cohort) |
| `membership.csv` | CSV | `checks.membership` | [preprocessing](../spec/preprocessing.md#cohort-resolution-per-cohort) |
| Patch coords | HDF5 | `io.h5` | [embeddings-and-patches](../formats/embeddings-and-patches.md) |
| Embeddings | HDF5 | `io.h5` | [embeddings-and-patches](../formats/embeddings-and-patches.md) |
| Outlines | polygon array + GeoJSON | `formats.outline` (+ `io.geojson`) | [outlines](../formats/outlines.md) |
| Transform | JSON | `config` (pydantic) | [wsi-transformation](../spec/wsi-transformation.md#artifacts-per-scan-per-variant) |
| Biopsy axis | JSON | `config` (pydantic) | [wsi-transformation](../spec/wsi-transformation.md#biopsy-axis-pca-line) |
| Bundle manifest | CSV | `checks.bundle` | [preprocessing](../spec/preprocessing.md#bundle-a-prepared-cohort) |
| Bundle metadata | JSON | `config` (pydantic) | [preprocessing](../spec/preprocessing.md) |
| Folds | CSV | `checks.folds` | [training](../spec/training.md#fold-assignment-the-generate_folds-rule) |
| Run record | JSON | `config` (pydantic) | [training](../spec/training.md#run-record-runjson-one-per-run) |
| `runs.parquet` | Parquet | `io.tables` | [training](../spec/training.md) / [reports](../design/11-reports.md) |
| BEAM | HDF5 | `formats.beam` | [BEAM format](../formats/beam.md) |
| Heatmap | PNG + GeoJSON | (produced by `histomil.heatmaps`) | [heatmaps](../design/08-heatmaps.md) |

---

## Scan manifest (pydantic, hierarchical)

Mirrors [`spec/data-ingestion.md`](../spec/data-ingestion.md#scan-manifest-yaml-json). The
loader flattens nested `metadata:` into per-level dicts (`dataset/patient/biopsy/scan`) carried
downstream.

```python
class Scan(BaseModel):    path: Path; metadata: dict = {}
class Biopsy(BaseModel):  scans: dict[str, Scan]; metadata: dict = {}   # key = stain
class Patient(BaseModel): biopsies: dict[str, Biopsy]; metadata: dict = {}
class Manifest(BaseModel):
    dataset_id: str; metadata: dict = {}
    patients: dict[str, Patient]
    # validators: stain ∈ vocabulary; path exists & OpenSlide-readable;
    # if registration enabled, every biopsy has the reference_stain
```

## Tabular contracts (`checks.py`)

The leakage-critical table invariants are explicit assertion functions in
`histomil.shared.checks` — clearer than declarative schemas for cross-row rules, and called by
both the producing rule and the contract tests. Column presence/dtypes are checked on read by
`io.tables.read_table(path, columns={...})`.

```python
# histomil/shared/checks.py
def check_membership(df):           # (dataset_id, patient_id, biopsy_id, role); role ∈ {development, holdout}; one role per patient
def check_folds(df, membership):    # (patient_id, fold, split∈{train,val,test}); holdout excluded; no patient in two splits of a fold
def check_labels(df):               # long form (dataset, patient, biopsy, label_name, label_value, label_type∈{continuous,binary,ordinal,categorical})
def check_bundle(df):               # bag_id unique; n_patches > 0 and == embedding row count; role ∈ {development, holdout}
def check_coords_match(emb, coords) # embedding row count + order == coords (the BEAM invariant)
```

These are exactly the cross-row invariants a single-column schema couldn't express anyway (no
patient in two folds, holdout in no fold, `n_patches` == embedding rows, `membership_hash`
matches), so collapsing to plain functions loses nothing and removes a dependency.

## HDF5

### coords

```text
coords        (N,2) int32     x,y top-left, level-0 frame
@patch_size   int             @level int   @mpp float
@source_variant str           @quartile (N,) int8   # 1–4; 0 unassigned
```

### embeddings

```text
coords        (N,2|4) int32   matches the coord file
embeddings    (N,D)   float32 RAW model output — no fitted normalization
@embedding_model_id @embedding_dim @patch_config_id @patch_size @mpp @source_variant
```

`shared.io.h5` enforces dtypes and the presence of every attribute on write, and refuses to
write an `embeddings` array whose row count ≠ `coords` (the
[bundle/eval invariant](../spec/evaluation.md#invariants)).

### BEAM (`shared.formats.beam`)

Exact layout from [`formats/beam.md`](../formats/beam.md). One BEAM per **(biopsy, sweep)** —
every `fold_seed × model_seed` model of one experiment-config run entry (`run_family`) writes its
own prediction/attention/stats into the same file, under `models/{run_id}/`:

```text
{biopsy_id}__{run_family}.beam.h5
  @ root attrs: format_version, biopsy/patient/dataset_id, run_family, model_ids,
                model_experiment, bundle_id, cohort_id, embedding_model_id, subset,
                evaluation_tag, membership_hash, git_commit, stain, source_variant,
                patch_config_id, patch_size, patch_resolution, quartile
  patches/coords (N,2|4) int   patches/size              # shared — one bag, every model
  outline/polygon (M,2) float  outline/quartiles/q0..q3  # shared, optional
  labels/      name → value (+type)                       # shared, when available
  embeddings   (N,D) float                                # shared, optional
  metadata/{dataset,patient,biopsy,scan}/                  # shared, forwarded manifest metadata
  models/
    {run_id}/                                              # one group per contributing model
      @ fold_seed, model_seed, checkpoints_used, git_commit,
        architecture_json, hyperparameters_json,
        target_normalization_json, metrics_json            # JSON-encoded RunRecord copy
      prediction
      attention/{raw,sigmoid,rank} (N,) float               # iff THAT model's attention architecture
```

`BeamWriter` guarantees: a model's `attention/` present **iff** that model's architecture is
attention-based (never zeros — a sweep may mix attention and non-attention models); `model_ids`
computed by the writer from what was written, never hand-supplied; `patches/coords` equal the
bundle embedding coords in the same order, shared by every model; each model's `prediction` in
label units (de-normalized using that model's own `target_normalization` when it was on); every
contributing model's `RunRecord` agrees on `run_family`/`bundle_id`/`cohort_id`/`membership_hash`/
`subset` (a mismatch is a hard error). `BeamReader` is the single entry point reporting and
heatmaps use — they depend on `histomil.shared`, never on `histomil.evaluation`.

## JSON artifacts

```python
# shared/config.py (transform model)
class Transform(BaseModel):
    variant: Literal["rigid","elastic"]
    affine: list[list[float]]          # 3×3, raw → variant
    reference_frame_size: tuple[int,int]
    displacement_field_ref: str | None = None   # elastic only

# shared/config.py (axis model)  — fields verbatim from the wsi-transformation spec
class BiopsyAxis(BaseModel):
    centroid: tuple[float,float]; direction: tuple[float,float]
    t_min: float; t_max: float; length_px: float; length_mm: float
    variance_ratio: float; quartile_cuts: list[float]   # len 5, monotonic

# shared/config.py (runs model) — run.json (flattened into runs.parquet rows)
# run_id = f"{run_family}__fs{fold_seed}__ms{model_seed}"; run_family is the
# experiment-config run entry id (e.g. "ki67_conch") shared by its whole seed sweep
# — what BEAM groups a sweep's contributing models by.
class RunRecord(BaseModel):
    run_id: str; run_family: str; model_experiment: str; bundle_id: str; cohort_id: str
    target: str; subset: str; seed_set: str; fold_seed: int; model_seed: int
    architecture: dict; hyperparameters: dict
    target_normalization: dict          # {enabled, per-fold mean/std | class_map}
    metrics: dict                       # {train,val,test: {metric: value}}
    checkpoints: dict[str, Path]
    membership_hash: str; git_commit: str
    started_at: datetime; finished_at: datetime
```

## GeoJSON exports

`shared.io.geojson` converts polygon arrays ↔ GeoJSON `Polygon`/`MultiPolygon`. Used for outline
viewing and for patch geometry carrying attention as feature properties. **Never** read back for
computation — GeoJSON is the view, binary is the source of truth
([data model](../design/02-data-model.md#storage-formats)).

## Bundle metadata (`metadata.json`)

```json
{ "cohort_id": "...", "membership_hash": "...", "embedding_model_id": "conch",
  "patch_config": {"patch_size":256,"resolution_mpp":0.5,"overlap":0.0},
  "source_variant": "raw",
  "metadata": {"dataset": {...}, "patient": {...}, "biopsy": {...}, "scan": {...}} }
```

The `metadata` block is the forwarded scan-manifest metadata grouped by level, so reports and
BEAM never re-join the source.
