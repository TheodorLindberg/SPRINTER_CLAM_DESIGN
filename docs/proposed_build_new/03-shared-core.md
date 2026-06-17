# 03 · The shared package (`histomil-shared`)

`histomil-shared` is the one distribution every stage depends on and that depends on no stage —
the concrete form of the design's "shared files," and the center of the
[dependency star](01-repository-layout.md#the-dependency-star). It holds **definitions and
contracts**; behaviour stays in the stage packages. Seven areas: **config**, **identifiers**,
**paths**, **IO**, **schemas + formats**, **model definitions**, **provenance/hashing**.
Signatures are illustrative of the boundaries, not finished code.

> The rule that decides what lands here: *if a piece would otherwise make two stage packages
> depend on each other, it belongs in `histomil-shared`.* That single rule keeps the graph a
> star instead of a web.

## config (`histomil.shared.config`)

`base.yaml` (in `components/shared/config/`) layers **under** every stage config — the rule in
[`docs/design/10-configuration.md`](../design/10-configuration.md). One loader merges and
validates into typed pydantic models.

```python
class Roots(BaseModel):            # base.yaml → roots:
    raw: Path; normalized: Path; processed: Path
    bundles: Path; results: Path; debug: Path
class Defaults(BaseModel):
    source_variant: Literal["raw","rigid","elastic"] = "raw"
    reference_stain: str = "HE"
class BaseConfig(BaseModel):
    roots: Roots; registries: dict[str, Path]; defaults: Defaults

def load(base: Path, *stage: Path) -> dict:        # deep-merge base then each stage (later wins)
def load_stage(model: type[T], base: Path, stage: Path) -> tuple[BaseConfig, T]:
    """Validate base into BaseConfig and the stage block into the stage's own model."""
```

- Each **stage package** defines its own config model (`WsiTransformationConfig`,
  `PreprocessingConfig`, …) and passes it to `load_stage` — so the shared loader knows the
  layering rule, the stage owns its schema.
- Registries (`cohorts.yaml`, `seeds.yaml`, in `components/shared/config/`) load into
  `CohortRegistry` / `SeedRegistry` models, used by `preprocessing` and `training` respectively.
- A config's content hash (`hashing.config_hash`) is recorded in provenance.

## ids (`histomil.shared.ids`)

All id construction/parsing in one module so the
[naming rules](../design/02-data-model.md#identifiers) exist once; stages never f-string an
id.

```python
def scan_id(biopsy_id, stain) -> str                       # "{biopsy}__{stain}"
def bundle_id(cohort, stain, source_variant, embedding_model) -> str
def bag_id(dataset_id, patient_id, biopsy_id, stain, patch_config_id, source_variant, embedding_model_id) -> str
def parse_bundle_id(s) -> BundleParts ; def parse_bag_id(s) -> BagParts   # round-trip
```

## paths (`histomil.shared.paths`)

The single authority turning `roots` + ids into concrete paths — what makes "change an output
path = one edit" hold. Every stage asks `Paths`, never composes strings. (Full template list as
in the previous draft: registered TIFF, transform, outline, axis, coords, embeddings, cache
object, membership, bundle dir, folds, run dir, runs.parquet, beam, report, debug.) Scan-level
artifacts are namespaced by `dataset/patient` because `biopsy_id`/`scan_id` are unique only
within a patient.

## io (`histomil.shared.io`) + formats (`histomil.shared.formats`)

Typed readers/writers so a format spec is enforced once. **IO** = generic typed access (HDF5,
GeoJSON, tables, WSI). **formats** = the cross-component *named* binary formats whose producer
and consumer live in different packages:

```python
# shared.io.h5
def write_coords(path, coords, *, patch_size, level, mpp, source_variant, quartile)
def write_embeddings(path, coords, embeddings, *, embedding_model_id, embedding_dim, ...)
# shared.io.wsi — one interface over OpenSlide (NDPI) + tifffile/zarr (OME-TIFF)
class WSIReader:
    def best_level_for_mpp(self, mpp) -> tuple[int, float]   # real metadata, never assumed 40×/20×
    def read_region(self, x, y, size, level) -> np.ndarray
# shared.formats.beam — written by evaluation, READ by heatmaps + reporting
class BeamWriter: ...        # see 06 for the full BEAM layout
class BeamReader: ...
# shared.formats.outline — polygon-array read/write (+ GeoJSON export via io.geojson)
```

Putting **BEAM** and **outline** read/write here is what lets `heatmaps` and `reporting` consume
evaluation's output without importing `histomil-evaluation`.

## schemas (`histomil.shared.schemas`)

`pandera` for tables, `pydantic` for hierarchical configs/manifests — the executable contracts
in [06](06-formats-and-schemas.md). They live in `shared` so training/evaluation validate what
preprocessing wrote against one definition, and so the test suite imports contracts without
heavy stage code.

## models (`histomil.shared.models`)

Model **definitions** (not training/inference behaviour) shared across stages:

- `models/embedding/` — the registry `id → (load(pretrained) -> nn.Module, NORM)`; used by
  `preprocessing.embed`. Ported from the old pipeline's clean `_MODEL_MODULES` pattern.
- `models/mil/` — the MIL heads (`clam`, `non_clam`, `regression`) behind a `MILModel` protocol
  (`forward(bag) -> prediction (+ attention?)`). Living here lets `evaluation` reconstruct the
  architecture from a run record to load a checkpoint **without** importing `histomil-training`.

`torch` is a `shared` *extra* (`histomil-shared[torch]`), so a report-only environment installs
`shared` without it.

## provenance & hashing

```python
# shared.provenance
class Provenance(BaseModel):
    git_commit: str; config_hash: str; membership_hash: str | None
    started_at: datetime; finished_at: datetime | None
def stamp() -> Provenance
# shared.hashing
def membership_hash(rows) -> str        # sha1 over canonical-sorted (dataset,patient,biopsy,role)
def cache_key(coords_row, patch_size, mpp, embedding_model_id, source_variant, aug) -> str
def config_hash(cfg) -> str
```

`membership_hash` and `cache_key` are defined once so freeze-and-detect-drift
([`spec/preprocessing.md`](../spec/preprocessing.md)) and the
[content-addressed cache](../formats/embeddings-and-patches.md#content-addressed-cache-key)
can never disagree between writer and checker.

## What `shared` deliberately excludes

- No VALIS, plotly, or Optuna — those are single-stage concerns, kept out so `shared` stays
  light and the star holds.
- No fitted statistics, ever — there is no code path here (or in the bundle writer) that
  persists a mean/std/threshold ([principle 4](README.md#guiding-principles-for-the-build)).
- No stage *behaviour* — only the definitions and contracts those behaviours read and write.
