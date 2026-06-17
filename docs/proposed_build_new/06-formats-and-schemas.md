# 06 · Formats & schemas

Every artifact in the pipeline as a concrete, executable schema, mapped to its format spec.
**All definitions live in `histomil-shared`** — that is precisely what lets the producing and
consuming stage packages stay independent (no cross-dep) while reading/writing against one
contract. Tabular artifacts get a `pandera` schema in `histomil.shared.schemas`; hierarchical
configs/manifests get a `pydantic` model; binary artifacts get a typed reader/writer in
`histomil.shared.io` (generic) or `histomil.shared.formats` (named cross-component formats). The
rule: a format is defined **once**, and both the producer and the test suite import that one
definition.

| Artifact | Format | Definition (`histomil.shared.…`) | Spec |
|---|---|---|---|
| Scan manifest | YAML/JSON | `schemas.manifest` (pydantic) | [data-ingestion](../spec/data-ingestion.md) |
| Raw label tables | CSV/Parquet | `schemas.labels` | [data-ingestion](../spec/data-ingestion.md#label-tables-optional) |
| Derived labels (long) | CSV | `schemas.labels` | [preprocessing](../spec/preprocessing.md#bundle-a-prepared-cohort) |
| `membership.csv` | CSV | `schemas.membership` | [preprocessing](../spec/preprocessing.md#cohort-resolution-per-cohort) |
| Patch coords | HDF5 | `io.h5` | [embeddings-and-patches](../formats/embeddings-and-patches.md) |
| Embeddings | HDF5 | `io.h5` | [embeddings-and-patches](../formats/embeddings-and-patches.md) |
| Outlines | polygon array + GeoJSON | `formats.outline` (+ `io.geojson`) | [outlines](../formats/outlines.md) |
| Transform | JSON | `schemas.transform` | [wsi-transformation](../spec/wsi-transformation.md#artifacts-per-scan-per-variant) |
| Biopsy axis | JSON | `schemas.axis` | [wsi-transformation](../spec/wsi-transformation.md#biopsy-axis-pca-line) |
| Bundle manifest | CSV | `schemas.bundle` | [preprocessing](../spec/preprocessing.md#bundle-a-prepared-cohort) |
| Bundle metadata | JSON | `schemas.bundle` | [preprocessing](../spec/preprocessing.md) |
| Folds | CSV | `schemas.folds` | [training](../spec/training.md#fold-assignment-the-generate_folds-rule) |
| Run record | JSON | `schemas.runs` | [training](../spec/training.md#run-record-runjson-one-per-run) |
| `runs.parquet` | Parquet | `schemas.runs` | [training](../spec/training.md) / [reports](../design/11-reports.md) |
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

## Tabular schemas (pandera)

```python
# shared/schemas/membership.py
Membership = DataFrameSchema({
    "dataset_id": Column(str), "patient_id": Column(str),
    "biopsy_id": Column(str), "role": Column(str, Check.isin(["development","holdout"])),
})
# shared/schemas/folds.py  — development patients only; holdout excluded (checked)
Folds = DataFrameSchema({
    "patient_id": Column(str),
    "fold": Column(int, Check.ge(0)),
    "split": Column(str, Check.isin(["train","val","test"])),
})
# shared/schemas/labels.py  — long form
DerivedLabels = DataFrameSchema({
    "dataset_id": Column(str), "patient_id": Column(str), "biopsy_id": Column(str),
    "label_name": Column(str), "label_value": Column(float, nullable=True),
    "label_type": Column(str, Check.isin(["continuous","binary","ordinal","categorical"])),
})
# shared/schemas/bundle.py  — manifest.csv
BundleManifest = DataFrameSchema({
    "bag_id": Column(str, unique=True),
    "dataset_id": Column(str), "patient_id": Column(str), "biopsy_id": Column(str),
    "stain": Column(str), "role": Column(str, Check.isin(["development","holdout"])),
    "embedding_path": Column(str), "n_patches": Column(int, Check.gt(0)),
})
```

Cross-row invariants that a single-column schema can't express (no patient in two folds; holdout
in no fold; `n_patches` == embedding row count; `membership_hash` matches) are `pandera`
dataframe-level `Check`s or a `validate_*` function beside the schema — reused by the rule and
the contract test.

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
@embedding_model_id @embedding_dim @patch_size @mpp @source_variant @augmentation_id
```

`shared.io.h5` enforces dtypes and the presence of every attribute on write, and refuses to
write an `embeddings` array whose row count ≠ `coords` (the
[bundle/eval invariant](../spec/evaluation.md#invariants)).

### BEAM (`shared.formats.beam`)

Exact layout from [`formats/beam.md`](../formats/beam.md):

```text
{biopsy_id}__{run_id}.beam.h5
  @ root attrs: format_version, biopsy/patient/dataset_id, run_id, model_experiment,
                bundle_id, cohort_id, embedding_model_id, checkpoints_used, subset,
                evaluation_tag, membership_hash, git_commit, stain, source_variant,
                patch_config_id, patch_size, patch_resolution, quartile
  patches/coords (N,2|4) int   patches/size
  attention/{raw,sigmoid,rank} (N,) float     # iff attention architecture
  outline/polygon (M,2) float  outline/quartiles/q0..q3
  prediction
  labels/      name → value (+type)           # when available
  embeddings   (N,D) float                    # optional; else referenced from bundle
  metadata/{dataset,patient,biopsy,scan}/     # forwarded manifest metadata, grouped by level
```

`BeamWriter` guarantees: `/attention` present **iff** architecture is attention-based (never
zeros); `patches/coords` equal the bundle embedding coords in the same order; `prediction` in
label units (de-normalized when `target_normalization` was on). `BeamReader` is the single entry
point reporting and heatmaps use — they depend on `histomil-shared`, never on
`histomil-evaluation`.

## JSON artifacts

```python
# shared/schemas/transform.py
class Transform(BaseModel):
    variant: Literal["rigid","elastic"]
    affine: list[list[float]]          # 3×3, raw → variant
    reference_frame_size: tuple[int,int]
    displacement_field_ref: str | None = None   # elastic only

# shared/schemas/axis.py  — fields verbatim from the wsi-transformation spec
class BiopsyAxis(BaseModel):
    centroid: tuple[float,float]; direction: tuple[float,float]
    t_min: float; t_max: float; length_px: float; length_mm: float
    variance_ratio: float; quartile_cuts: list[float]   # len 5, monotonic

# shared/schemas/runs.py — run.json (flattened into runs.parquet rows)
class RunRecord(BaseModel):
    run_id: str; model_experiment: str; bundle_id: str; cohort_id: str
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
