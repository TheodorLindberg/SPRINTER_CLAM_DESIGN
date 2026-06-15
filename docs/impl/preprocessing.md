# Impl · Dataset Preprocessing

Implementation notes for [Stage 3](../design/05-dataset-preprocessing.md). [Overview](../design/05-dataset-preprocessing.md) · [Specification](../spec/preprocessing.md) · **Implementation**.

Reference libraries: **shapely** (geometry), **tifffile/zarr** or **openslide** (WSI I/O), **PyTorch + timm/HF** (embedding), **h5py** (storage), **pandas/pyarrow** (manifests).

> Design-level notes, not a prescriptive recipe. The contract is the [preprocessing spec](../spec/preprocessing.md).

## Cohort resolution

Runs first, before any bundle or fold. The members are expanded per dataset source (`patients: all`, or an explicit list), each patient is tagged `holdout` (from the explicit list or a deterministic `fraction` + `seed`) or otherwise `development`, and the set is expanded to biopsy level via the scan manifest. After [validation](../spec/preprocessing.md#validation-hard-errors) it is written as `membership.csv` plus a `membership_hash` over the canonicalised rows, and a cohort HTML report is built.

- The frozen membership + hash is what [`assemble_bundle`](#bundle-assembly) and [`generate_folds`](training.md#fold-generation-the-generate_folds-rule) read; a hash change flags dependents stale.
- The report reuses the report toolkit (Plotly + `reports.css`): composition by dataset × role, label histograms per role, and provenance.

## Patch generation

Slide a grid over the outline's bounding box with stride `patch_size × (1 − overlap)`, and keep a patch when its intersection with the tissue outline covers enough of its area. Each kept patch is tagged with its [biopsy-axis](wsi-transformation.md#biopsy-axis-pca-line) quartile (project the patch centre onto the axis). Coordinates and attributes are written to HDF5.

- Coordinates are level-0; pixel reads happen at the configured `level` / `mpp`.
- Patches near a quartile boundary may be dropped with a small buffer to avoid cross-quartile leakage.

## Embedding

Load the registry model and its fixed normalization, open the slide (OME-TIFF via tifffile/zarr, NDPI via openslide), then read patches in batches at the configured level, apply the model's normalization, embed, and append. Coordinates, embeddings, and attributes are saved per scan to HDF5.

- Device: respect `CUDA_VISIBLE_DEVICES`; otherwise pick the GPU with the most free memory.
- HuggingFace offline / SSL workarounds as needed on HPC.

!!! note "Keep the GPU fed"
    The bottleneck here is patch **decoding** (random-access WSI region reads + JPEG tile decode), not the model forward — a naive read-then-embed loop leaves the GPU idle. Decode and normalize in a CPU worker pool that prefetches into a queue while the GPU consumes large batches (pinned memory), so I/O overlaps compute. Order reads along the slide's native tile grid — or read one larger region and sub-crop several patches — so decoded tiles are reused and access stays sequential rather than random.

!!! note "Input normalization is fixed per model"
    The image mean/std is a **predetermined constant** carried in each registry entry, never fitted on the cohort. Most models use ImageNet stats (`[0.485, 0.456, 0.406]` / `[0.229, 0.224, 0.225]`); CLIP-based models (e.g. CONCH) use their own. Keeping it per-model stops one exception silently mis-normalizing.

### Embedding models

Each `embedding_model_id` maps to a registry module exposing a loader (`load(pretrained) -> nn.Module`) and a fixed `NORM`. Supported encoders (the model page / repo holds the exact load snippet):

| Model | Source | Dim | Norm | Requires |
|---|---|---|---|---|
| `uni2_h` | [MahmoodLab/UNI2-h](https://huggingface.co/MahmoodLab/UNI2-h) (timm hub) | 1536 | ImageNet | HF token + gated access |
| `conch` | [MahmoodLab/CONCH](https://github.com/mahmoodlab/CONCH) (CLIP `encode_image`) | 512 | CLIP | HF token + gated access + `conch` package |
| `gigapath` | [prov-gigapath](https://huggingface.co/prov-gigapath/prov-gigapath) tile encoder (timm hub) | 1536 | ImageNet | HF token + gated access |
| `tenpercent_resnet18` | [ozanciga/self-supervised-histopathology](https://github.com/ozanciga/self-supervised-histopathology) (resnet18, `fc=Identity`) | 512 | ImageNet | local checkpoint (not on HF) |

!!! note "Access requirements"
    `uni2_h`, `conch`, and `gigapath` are **gated** on HuggingFace: accept the terms on each model page with an account that has access, and set `HF_TOKEN` before the first download. `conch` additionally needs its `conch` package. `tenpercent_resnet18` is **not** on HF — download its checkpoint from the linked repo.

To add a model: create a registry module (loader + `NORM`) and reference it by `embedding_model_id`.

### Content-addressed cache

The [cache key](../spec/preprocessing.md#cache-key) is computed per scan-config; a run diffs the requested coordinates against what's cached, **embeds only the delta**, copies reused rows by `(x, y)` index, and reassembles them in the requested order. For how this incremental cache stays consistent with Snakemake's file-based DAG, see [Embedding cache vs. the DAG](workflow.md#embedding-cache-vs-the-dag).

### Augmentation

Each configured augmentation is applied to the patch image **before** embedding, then cached under its own `augmentation_id`; `n_variants` sets how many augmented copies per patch. The proven baseline is rotation (90/180/270) and colour jitter (brightness/contrast/saturation/hue); flips and stain-space (HED) jitter slot into the same transform pipeline. The foundation model runs here, never in the training loop.

Read each base patch from the slide **once**, generate all `n_variants` augmentations in memory, and embed them together — never re-open the slide per augmentation (the WSI read is the expensive part, not the transform). Each variant is still written to its own `augmentation_id` cache, so the embeddings stay independently addressable.

## Bundle assembly

Resolve the cohort (frozen members + roles), then for each `(patient, biopsy, stain)` locate the cached embedding — skipping and logging when a stain is missing for that patient — mint its `bag_id`, and symlink the embedding into the bundle with a manifest row. The bundle gets a manifest, a labels file, and a metadata file.

- **Symlink** embeddings/outlines by default (cheap); copy only when a bundle must be portable.
- **Labels** — merge the configured `labels.sources` tables on `(dataset, patient, biopsy)`, then compute the **derived labels** (mean, max, threshold-to-binary, …). Written to `labels.csv` in **long form** (one row per biopsy × label name); config-driven and extendable. Omit entirely for label-free bundles.
- Forward the scan-manifest metadata columns (with the membership hash) into the metadata file so they reach reports / BEAM.
- Emit nothing fitted — normalization happens at training time.
