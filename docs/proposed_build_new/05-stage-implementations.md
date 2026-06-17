# 05 · Stage implementations

Module-by-module build notes for stages 2–6 (Stage 1 is the user bridge against
[`spec/data-ingestion.md`](../spec/data-ingestion.md)). Each maps a Snakemake rule to a
package module, names the engine it wraps, and pins the contract it must satisfy. Signatures
are illustrative; the spec pages are authoritative.

Reuse markers come from `OLD_PIPELINE_REVIEW.md` (repo root, outside the docs site): **[port]** = take
largely as-is, **[new]** = build fresh, **[adapt]** = port the core, rebuild the glue.

---

Module paths are `histomil.<stage>.<module>`, with shared definitions/contracts under
`histomil.shared.*`.

## Stage 1 · Ingestion bridge (`ingestion/`)

Outside Snakemake. A per-dataset script emits the scan files + a
hierarchical scan manifest (YAML/JSON) + optional label tables, conforming to
[`spec/data-ingestion.md`](../spec/data-ingestion.md). The build ships
`ingestion/sahlgrenska_2018.py` as the reference and a shared
`histomil.shared.manifest.validate(path)` the bridge calls to self-check. The pipeline
reads the manifest, never the layout.

---

## Stage 2 · WSI Transformation (`histomil.wsi_transformation`)

Contract: [`spec/wsi-transformation.md`](../spec/wsi-transformation.md). Registration +
outlines + QC happen in **one rule** (`register`) because VALIS already segments tissue and
holds the transforms; `biopsy_axis` is a separate geometry rule.

| Module | Rule | Wraps / does | Output |
|---|---|---|---|
| `register.py` **[port]** | `register` | VALIS rigid (feature-based) + elastic (displacement field); fit `raw→variant` affine, persist registrar | OME-TIFF per variant + `transform.json` |
| `outlines.py` **[adapt]** | `register` | tissue mask → contours (OpenCV) → simplify → polygon array; `valis` (default) or `hsv_otsu` | `…/outlines/{scan}__{variant}.geojson` (method-agnostic path) + polygon array |
| `intersection.py` **[adapt]** | `register` | cross-stain intersection from VALIS elastic overlap (or shapely intersect of hsv_otsu) | polygon array, per scan |
| `qc.py` **[port]** | `register` | low-res PNG: outline on tissue; HE thumbnail with all outlines + intersection | QC PNGs |
| `biopsy_axis.py` **[port]** | `biopsy_axis` | PCA on raw tissue pixels → principal axis, projection extent, quartile cuts; skeleton fallback when `variance_ratio` low | `…/axis/{scan}.json` |

Key build points:

- **Persist the transform** (`transform.json`: affine 3×3 + reference-frame size; elastic =
  displacement-field handle + rigid prefix). Heatmaps render on registered underlays later, so
  the `raw→variant` mapping must be saved, not just applied — the warning in
  [`impl/wsi-transformation.md`](../impl/wsi-transformation.md).
- **Axis computed in the raw frame** (the frame training patches come from) so a raw patch's
  quartile is well-defined; registered-frame copies are derived for visuals only.
- **Method comparison is debug-only**: `debug_compare_methods: true` additionally runs the other
  method and writes its outline + side-by-side overlay under `roots.debug`; nothing downstream
  reads it, and no stage branches on tissue method (one authoritative outline per `(stain,
  variant)`).
- Invariants to assert ([spec](../spec/wsi-transformation.md#invariants)): rigid transform
  applied to raw outline ≈ rigid outline; intersection ⊆ every stain's elastic outline; axis
  `direction` unit-norm; `quartile_cuts` monotonic, four equal segments; `variance_ratio ≥ 0.9`
  flagged.

---

## Stage 3 · Dataset Preprocessing (`histomil.preprocessing`)

Contract: [`spec/preprocessing.md`](../spec/preprocessing.md). Cohort resolution runs first
and gates everything.

### `cohort.py` **[adapt]** — `resolve_cohort`

Expand `cohorts.yaml` members per dataset source (`all` or explicit list), tag each patient
`holdout` (explicit list or deterministic `fraction`+`seed`) else `development`, expand to
biopsy level via the manifest, **validate** (hard errors + warnings per spec), then write
`processed/cohorts/{cohort}/membership.csv` + a `membership_hash` over canonical-sorted rows, and
emit `results/reports/cohorts/{cohort}.html` via the [report toolkit](07-reports.md).

```python
def resolve_cohort(cfg, registry, manifests, labels) -> Membership
# Membership: rows[(dataset_id, patient_id, biopsy_id, role)] + membership_hash
```

Hard errors: member missing from manifest; holdout not a member; patient with >1 role;
non-reproducible fraction split; pooled target `name`/`type` mismatch. Warnings: member missing a
requested stain (no bag); dev patient missing the target value.

### `labels.py` **[port]** — `derive_labels`

Merge `labels.sources` tables on the **full** `(dataset, patient, biopsy)` key, compute derived
labels (`mean`, `max`, threshold-to-binary, …) per `preprocessing.yaml → labels.derived`, write
**long form** `(dataset, patient, biopsy, label_name, label_value, label_type)`. Quartile scores
averaged here (carried metadata, not geometry). Config-driven and extendable. Ports the old
Ki-67 aggregation logic.

### `patching.py` **[port]** — `patch_coords`

Grid over the outline bbox with stride `patch_size × (1 − overlap)`; keep a patch when its
intersection with the tissue outline covers enough area; tag each with its biopsy-axis quartile
(project centroid onto axis); optional boundary buffer to avoid cross-quartile leakage. Write
coords HDF5 ([06 schema](06-formats-and-schemas.md#coords)). Coordinates are level-0; pixel reads
happen at the configured `level`/`mpp`.

### `embed.py` **[port]** — `embed`

The ported embedding engine: the `histomil.shared.models.embedding` registry loads
`(load(pretrained), NORM)`; `shared.io.wsi.WSIReader` reads patches in batches at the target mpp;
apply the model's **fixed** normalization
(per-model constant, never fitted); embed on GPU; write per-scan HDF5.

- **Keep the GPU fed**: CPU worker pool decodes+normalizes into a queue while the GPU consumes
  large pinned batches; order reads along the native tile grid (or read one region, sub-crop
  several patches). The bottleneck is patch decode, not the forward.
- **File-level cache**: write one HDF5 per `(scan, variant, embedding_model, patch_config)` at a
  path that encodes those identifiers, rows in coordinate order. The path is the cache: Snakemake
  skips a configuration whose file exists and rebuilds only changed ones — see
  [04 · file-level cache](04-snakemake-integration.md#file-level-embedding-cache).

### `bundle.py` **[new]** — `assemble_bundle`

For each `(patient, biopsy, stain)` in the frozen membership, locate the cached embedding
(skip+log a missing stain), mint `bag_id`, symlink the embedding (copy only if portable), and
write a manifest row. Emit `manifest.csv`, `labels.csv` (long form; absent if label-free),
`metadata.json` (cohort_id, membership_hash, embedding_model_id, patch_config, source_variant +
forwarded scan-manifest metadata by level). **No fitted statistics** — the writer has no field
for them ([principle 4](README.md#guiding-principles-for-the-build)). Bundle schema in
[06](06-formats-and-schemas.md#tabular-contracts-checkspy).

---

## Stage 4 · Model Training (`histomil.training`)

Contract: [`spec/training.md`](../spec/training.md).

### `folds.py` **[port]** — `generate_folds`

Stratified k-fold (quantile bins for regression) over **development** patients, seeded by
`fold_seed`; fold *i*: `test=i`, `val=i+1`, `train=rest`; patient-level (all bags share a fold);
fall back to a shuffled split when a class has `< n_folds` patients. Write
`results/folds/{seed_set}/{target}/{fold_seed}.csv` — keyed by `target` only when stratified
(stratifying makes the split target-dependent). Generated **once**; every run reads the CSV and
never re-derives from a seed.

### `bagstore.py` **[new]** — load-once bag store

The bundle's embeddings are static, so load all bags into RAM once (or memory-map the H5) and
share across every `fold_seed × model_seed` run and across evaluation. Balancing/sampling select
**indices** into this store; optionally pre-pack into one contiguous array + offset index for a
few large reads. Avoids the training analog of the per-patient I/O trap.

### `trainer.py` **[adapt]** — `train_run`

Per fold: if `target_normalization` (default on) fit target mean/std on the **train fold only**
and store it (else train in raw label units); MIL forward (bag → pooling → prediction); loss +
optimizer per family; early-stop on val; log per-epoch train/val; evaluate the `test` split;
save checkpoint + history. Emit a [`run.json`](06-formats-and-schemas.md#json-artifacts).

Architecture families live in **`histomil.shared.models.mil`** (shared with evaluation, which
must reconstruct them to load checkpoints — [see 03](03-shared-core.md#models-histomilsharedmodels)):

| Family | Module | Status | Notes |
|---|---|---|---|
| `clam` | `shared.models.mil.clam` | **[port]** | `clam_sb`/`clam_mb`; classification (bin continuous targets); exposes attention. Feeds bags **from the manifest**, no symlink hacks. |
| `non_clam` | `shared.models.mil.non_clam` | **[adapt]** | mean-pool baseline, no attention. |
| `regression` | `shared.models.mil.regression` | **[new]** | continuous-target MIL head (Huber/MSE). |

The **training loop** that drives them stays in `histomil.training.trainer`; only the model
*definition* is shared.

### `balancing.py` **[port]** — per fold, train split only

Classification: class weights or weighted/over/under-sampling. Regression: quantile-bin
balancing. Never fit on val/test ([no-leakage rule](../design/05-dataset-preprocessing.md)).

### `aggregate.py` **[new]** — `aggregate_runs`

Collect all `run.json` → `results/runs.parquet` (one row per run = flattened tags + headline
metrics + paths). Round-trips: one run dir ↔ one row.

### `hpo.py` **[new]** — `hpo`

Optuna study (TPE) over `hpo.space`, objective = mean val metric across a CV subset; promote best
`promote_top_n` into a seed sweep. Outputs segregated under
`results/experiments/{name}/hpo/` with their own index; retention per
`reports.yaml → hpo.keep_checkpoints` (default top-N).

---

## Stage 5 · Evaluation (`histomil.evaluation`)

Contract: [`spec/evaluation.md`](../spec/evaluation.md). One BEAM per `(biopsy, run)`.

### `routing.py` **[new]** — checkpoint routing (the crux)

| `subset` | Rule |
|---|---|
| `development` | each patient scored by its **out-of-fold** checkpoint (the fold where it was `test`) |
| `holdout` | scored by **all** fold checkpoints, aggregated (default mean); record contributors |
| `all` | single full-data checkpoint |

### `infer.py` **[adapt]** — `infer`

Organised **per model, not per patient**: load each checkpoint **once**, batch through every
patient it owns; reuse the load-once `bagstore`. Loading checkpoints once **and** embeddings once
is what keeps eval off the I/O floor. Validate run stain == bundle stain first.

### `aggregate.py` **[new]** + `shared.formats.beam` — `aggregate_beam`

De-normalize each checkpoint's prediction with its own stored stats **before** combining
(holdout = mean across checkpoints; dev/all = single checkpoint). Write one
`{biopsy_id}__{run_id}.beam.h5` via `histomil.shared.formats.beam.BeamWriter` — the writer lives
in `shared` so heatmaps and reporting read BEAM without importing `histomil.evaluation`
([06 BEAM schema](06-formats-and-schemas.md#beam-sharedformatsbeam)): provenance
attrs, `patches/coords` in the **raw frame** in model-fed order, prediction, attention transforms
(`raw`/`sigmoid`/`rank` — **only** for attention models, never faked zeros), outline + quartiles,
true label when present. HDF5 is appendable for later enrichment. Attention extraction logic
ported from the old `extract_attention.py`.

---

## Stage 6 · Heatmaps (`histomil.heatmaps`)

Contract: [`docs/design/08-heatmaps.md`](../design/08-heatmaps.md).

| Module | Does |
|---|---|
| `warp.py` **[new]** | push raw-frame BEAM coords through the chosen variant's `transform.json` before overlay — skipping this silently misplaces every patch |
| `render.py` **[port]** | attention (default `rank`) → perceptually-uniform colormap overlay on the chosen-variant WSI; mask non-tissue, draw outline, scale bar from mpp, provenance caption, colorbar → PNG |
| `geojson.py` **[port]** | patch polygons carrying attention as feature properties → TissUUmaps GeoJSON layer |

Renders + the TissUUmaps layer port the old `make_attention_tmap.py`, emitting GeoJSON instead of
`.tmap`. Cross-stain overlays exploit elastic alignment (attention from one stain on another's
morphology).

---

## Cross-stage: how a stage module is shaped

Every stage module follows the same skeleton so they are uniformly testable and CLI-callable:

```python
def run(cfg: StageConfig, paths: Paths, *inputs) -> Outputs:
    prov = histomil.shared.provenance.stamp()
    ...                      # pure logic; all IO via shared.io, all paths via `paths`
    return outputs           # validated against a shared.schemas contract before write
```

The component's CLI command and its Snakemake rule both call `run`; neither contains logic, and
neither reaches into a sibling component. This is what makes "each stage runs and updates
independently" real and keeps Snakemake a thin orchestrator.
