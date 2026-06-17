# Old pipeline review — `clam_old_pipeline`

A migration reference comparing the existing CLAM pipeline to the new design. **This file is intentionally outside `docs/`** — the published design docs reason only about the new pipeline and do not reference the old one. Use this to decide what to port and what to redesign.

> Source reviewed: `clam_old_pipeline/pipeline/` (Snakemake workflow, scripts, configs) plus its `README.md` / `CLAUDE.md`.

---

## 1. What the old pipeline is

A Snakemake pipeline that turns prostate-cancer WSIs into CLAM-ready MIL bags and trains/evaluates CLAM, on Alvis HPC (SLURM + Apptainer). Three core stages plus side-flows:

```
extract_coords → embed → (discover_quartiles checkpoint) → make_bags
              → make_flat_bags + make_clam_splits → [external CLAM training]
              → attention extraction → TissueMAPS .tmap visualisation
```

Side Snakefiles: `Snakefile_ki67_predict` (U-Net Ki-67 quantification → labels), `Snakefile_pca_quartiles` (data-driven quartile boundaries), `Snakefile_attention` (post-training attention heatmaps).

Key characteristics:

- **Two separate entrypoints**: `Snakefile` (pre-registered OME-TIFF, shared coordinate space) and `Snakefile_raw.smk` (raw NDPI, per-stain coordinates) — with duplicated `rules/raw/*` versions of coords/embed/regions/visualize.
- **Single dataset**: patients are discovered as `patient_N/` directories under one `dataset_dir`; identity is `patient_N` only.
- **Quartile-centric**: hand-drawn `Q1–Q4` GeoJSON polygons per patient; patches assigned to a quartile by Shapely area overlap; `mode: whole` unions them.
- **Registration is upstream/external**: the registered pipeline assumes OME-TIFFs are already aligned (HE defines the shared coord system); `docs/patient_symlink` stages them.
- **Splits**: patient-stratified k-fold (`StratifiedKFold` over patients); per fold `test=fold_i`, `val=fold_i+1`, `train=rest`. **No locked holdout** beyond the rotating CV test fold. One split seed; **no model-seed / fold-seed sweep**.
- **Config sprawl**: `config/{base,old,raw,new_reg,rescomp}/…` with many near-duplicate per-stain files; HPO is done by copying configs.
- **CLAM integration via filename + symlink hacks**: bags are bare `(N,D)` tensors; everything is encoded in a `slide_id` string `{patient}__{stain}__{patch_method}__{model}__{quartile}`; a `tumor_vs_normal_resnet_features` symlink tricks CLAM into finding bags.

---

## 2. Old → new mapping

| Old pipeline | New design |
|---|---|
| `patient_N` under one `dataset_dir` | `(dataset_id, patient_id)`; **cohorts** pool multiple dataset sources |
| Registered vs raw — two Snakefiles | One pipeline; **`source_variant`** (`raw`/`rigid`/`elastic`) as a first-class axis |
| Registration assumed upstream | **Stage 2 · WSI Transformation** (registration + outlines) is explicit |
| Q1–Q4 GeoJSON drives bag structure | Quartiles are **optional outline subdivision + metadata**; labels are biopsy-level |
| `embeddings.pt` + `coords_hash` incremental update | **Content-addressed embedding cache** (generalises the hash idea) |
| Bare `(N,D)` bag tensors, no metadata | **Bundle** = prepared cohort with a manifest + forwarded metadata |
| `slide_id` string encodes everything | Stable ids + **manifest columns** (no meaning in filenames) |
| dataset.csv + splits baked at prep time | Folds deferred to training; **seed sweep** (`seeds.yaml`) |
| Patient-stratified k-fold, no holdout | **development / holdout roles**; locked holdout, scored once |
| HPO by copying configs | Separate **`hpo.yaml`**, segregated outputs, top-N promotion |
| TissueMAPS `.tmap` viz, "wonky" attention | **BEAM** result format + Stage 6 heatmaps + TissUUmaps GeoJSON |
| Scattered configs | `base.yaml` + cohorts + seeds + per-stage + `model_experiment` (defaults + runs) |

---

## 3. What to REUSE (it's good — port largely as-is)

| Component | Old location | Why keep it |
|---|---|---|
| **Embedding engine** | `embed_wsi.py`, `embed_utils.py` | Streams OME-TIFF as zarr (no full decompress), batched GPU embedding, device selection. Solid. |
| **Model registry** | `workflow/scripts/models/*` + `_MODEL_MODULES` | Clean `load(pretrained)->nn.Module` + `NORM` pattern (resnet/uni2_h/conch/gigapath). Maps directly to `embedding_model_id`. |
| **Incremental re-embed / hashing** | `update_embeddings.py` (`coords_hash`) | The MD5-of-coords + reuse-matching-rows logic is exactly the seed of the content-addressed cache. |
| **Patch extraction** | `extract_coords.py` | Grid patching + Shapely quartile assignment + boundary-buffer exclusion. Reuse; feed from outlines. |
| **Bag construction** | `make_bags.py`, `make_patient_bag.py` | Filter embeddings → bag. Reuse, but attach manifest/metadata. |
| **Split logic** | `make_splits.py` | Patient-stratified k-fold + per-patient/per-quartile label auto-detect. Reuse the stratification; extend with holdout + seeds. |
| **Ki-67 label derivation** | `Snakefile_ki67_predict`, `ki67_aggregate_labels.py` | U-Net quantification → per-quartile hotspot → binning. This is the label-processing component. |
| **PCA quartile generation** | `Snakefile_pca_quartiles` | Data-driven outline subdivision; drop-in GeoJSON. Keep as an outline option. |
| **Attention extraction** | `extract_attention.py` | Feeds BEAM attention arrays. |
| **TissueMAPS rendering** | `make_patch_tmap.py`, `make_attention_tmap.py` | Reuse the rendering; emit GeoJSON layers for TissUUmaps. |
| **HPC orchestration** | `util/slurm/*`, cluster-generic profile, `container:` directives | The SLURM + Apptainer wiring is real, hard-won infra. Reuse. |
| **Checkpoint-for-dynamic-DAG** | `discover_quartiles` checkpoint | The Snakemake pattern for outputs unknown until runtime. |

---

## 4. What to UPDATE / redesign

1. **Unify registered + raw into one workflow** keyed on `source_variant`. The duplicated `rules/raw/*` and the second Snakefile are the biggest source of drift — collapse them.
2. **Consolidate configs.** Replace `config/{old,raw,new_reg,rescomp}` sprawl with `base.yaml` + registries (`cohorts`, `seeds`) + per-stage configs + `model_experiment` (defaults + explicit runs). HPO moves to its own `hpo.yaml`.
3. **Introduce dataset_id + cohorts.** Generalise `patient_N`-only identity to `(dataset_id, patient_id)` so multiple sources can be pooled. Add development/holdout **roles** and a **locked holdout** the CV never touches.
4. **Add the seed sweep + split registry.** Replace the single k-fold seed with independent fold-seed × model-seed variation; record splits against a frozen cohort-membership hash.
5. **Replace filename-encoded `slide_id` with manifests.** Keep stable ids; move stain/patch/model/quartile/role into manifest columns. Retire the `tumor_vs_normal_resnet_features` symlink and flat-bag naming hacks.
6. **Make bundles self-contained + carry metadata.** Old bags drop all metadata; the new bundle keeps a manifest + forwards entity metadata into the bundle and BEAM.
7. **Standardise results into BEAM + a runs index + HTML reports.** Replace the ad-hoc `.tmap`-only outputs and "wonky" attention path.
8. **Decouple quartiles from the core.** Treat quartile as optional geometric subdivision + carried metadata, not the unit that defines bags.
9. **Defer folds out of preprocessing.** Old bakes `splits_{i}.csv` at prep time; the new bundle stays fold-agnostic so one bundle feeds the whole sweep.

---

## 5. Gaps the new design fills (absent in old)

- Locked **holdout** cohort and the development/holdout/all **subset** model.
- **Multi-dataset** cohorts.
- **Seed sweep** and a named **split registry**.
- A real **HPO** framework with segregated outputs.
- **Self-contained bundles**, manifests, and end-to-end **provenance/metadata** forwarding.
- A unified **reports** layer (`runs.parquet`, faceted HTML, data export).
- A documented result format (**BEAM**).
- A single, layered **config** model.

---

## 6. Migration notes / watch-outs

- **CLAM coupling**: the old pipeline targets the bundled `clam/CLAM` repo and its CSV/symlink conventions. The new "training function" abstracts model families (regression / CLAM / non-CLAM); plan an adapter so CLAM still consumes bundle bags without the symlink hacks.
- **Labels-out-of-dataset.csv**: old keeps labels only in the labels CSV (CLAM joins at train time). New bundles carry all labels and pick a `target` per experiment — port the join logic accordingly.
- **SSL / offline HF**: old disables SSL and uses `HF_HUB_OFFLINE` on Alvis. Keep these container workarounds when porting the embedding engine.
- **Quartile boundary buffer**: the leakage-prevention exclusion near quartile borders is worth preserving if quartile subdivision is used.
- **Validation**: `validate_embeddings.py` (coords ↔ embeddings xy match) is a good acceptance-test pattern to carry into the new validation checks.