# Impl · Dataset Preprocessing

Recipe for [Stage 3](../design/05-dataset-preprocessing.md). [Overview](../design/05-dataset-preprocessing.md) · [Specification](../spec/preprocessing.md) · **Implementation**.

Reference libraries: **shapely** (geometry), **tifffile/zarr** or **openslide** (WSI I/O), **PyTorch + timm/HF** (embedding), **h5py** (storage), **pandas/pyarrow** (manifests).

## Patch generation

```python
step = round(patch_size * (1 - overlap))         # stride
for (x, y) in grid(outline.bounds, step):
    patch_poly = box(x, y, x+patch_size, y+patch_size)
    if patch_poly.intersection(outline).area / patch_poly.area >= min_overlap:
        t = project_to_axis(center(x, y), biopsy_axis)   # Stage 2 axis
        quartile = quartile_of(t, biopsy_axis.quartile_cuts)
        emit(x, y, quartile)
```

- Coordinates are level-0; reads happen at the configured `level`/`mpp`.
- Patches near a quartile boundary may be dropped with a small buffer to avoid cross-quartile leakage.
- Write `coords` + attrs to HDF5.

## Embedding

```python
model = load_model(embedding_model_id)            # registry entry: load(pretrained)->nn.Module + NORM
slide = open_slide(wsi_path)                       # OME-TIFF→tifffile/zarr; NDPI→openslide
for batch in batched(coords):
    imgs = [read_patch(slide, x, y, patch_size, level) for x, y in batch]
    imgs = normalize(imgs, model.NORM)             # fixed, predetermined per model (never data-fitted)
    emb  = model(to_tensor(imgs))                  # (B, D)
    append(emb)
save_h5(coords, embeddings, attrs)
```

- Device: respect `CUDA_VISIBLE_DEVICES`; else pick the GPU with most free memory.
- HuggingFace offline / SSL workarounds as needed on HPC.

!!! note "Input normalization is fixed per model"
    The image mean/std is a **predetermined constant** carried in each registry entry's `NORM`, never fitted on the cohort. Most models share ImageNet stats (`[0.485, 0.456, 0.406]` / `[0.229, 0.224, 0.225]`); CLIP-based models (e.g. CONCH) use their own. Keep it per-model so that one exception doesn't silently mis-normalize.

### Embedding models

Each `embedding_model_id` maps to a registry module exposing `load(pretrained) -> nn.Module` and a fixed `NORM`. Supported encoders (the model page / repo holds the load snippet):

| Model | Load | Dim | Norm | Requires |
|---|---|---|---|---|
| `uni2_h` | `timm.create_model("hf-hub:MahmoodLab/UNI2-h", …)` — [card](https://huggingface.co/MahmoodLab/UNI2-h) | 1536 | ImageNet | HF token + gated access |
| `conch` | `conch.open_clip_custom.create_model_from_pretrained("conch_ViT-B-16", "hf_hub:MahmoodLab/conch")` → use `encode_image` — [card](https://huggingface.co/MahmoodLab/CONCH) | 512 | CLIP | HF token + gated access + `pip install conch` |
| `gigapath` | `timm.create_model("hf_hub:prov-gigapath/prov-gigapath", …)` (tile encoder) — [card](https://huggingface.co/prov-gigapath/prov-gigapath) | 1536 | ImageNet | HF token + gated access |
| `tenpercent_resnet18` | torchvision `resnet18` + the repo checkpoint, `fc=Identity` — [ozanciga/self-supervised-histopathology](https://github.com/ozanciga/self-supervised-histopathology) | 512 | ImageNet | local `tenpercent_resnet18.ckpt` from the repo (no HF) |

!!! note "Access requirements"
    `uni2_h`, `conch`, and `gigapath` are **gated** on HuggingFace: accept the terms on each model page with an account that has access, and set `HF_TOKEN` in the environment before the first download. `conch` additionally needs its `conch` package. `tenpercent_resnet18` is **not** on HF — download its checkpoint from the linked repo.

To add a model: create a registry module (`load(pretrained) -> nn.Module` + `NORM`) and reference it by `embedding_model_id`.

### Content-addressed cache

Compute the [key](../spec/preprocessing.md#cache-key) per scan-config; diff requested vs cached coords; **embed only the delta**, copy reused rows by `(x,y)` index, reassemble in requested order. (Generalises the old `coords_hash` incremental re-embed.)

### Augmentation

Each configured augmentation — **rotation** (90/180/270) and **colour jitter** (brightness/contrast/saturation/hue) are the proven baseline — is applied to the patch image **before** embedding, then cached under its own `augmentation_id`. `n_variants` sets how many augmented copies per patch. Flips and stain-space (HED) jitter slot into the same `torchvision`-style transform pipeline. The foundation model runs here, never in the training loop.

## Bundle assembly

```python
members = resolve_cohort(cohort_id)               # frozen → (dataset, patient, role)
hash_   = membership_hash(members)
for (patient, biopsy, stain) in members × stains:
    emb = cache_path(scan, source_variant, embedding_model_id)
    if not exists(emb): continue                  # stain missing for this patient — log, skip
    bag_id = make_bag_id(...)
    symlink(emb, bundle/embeddings/f"{bag_id}.h5")
    rows.append({bag_id, biopsy_id, patient_id, dataset_id, stain,
                 role, embedding_path, n_patches})
write_csv(manifest, rows)
write_csv(labels, all_labels)                     # omit if label-free
write_json(metadata, {cohort_id, membership_hash: hash_, ...forwarded scan metadata})
```

- **Symlink** embeddings/outlines by default (cheap); copy only when a bundle must be portable.
- **Derived labels** (`mean`, `max`, threshold-to-binary, …) are computed here from raw scores and written to `labels.csv`; the set is config-driven and extendable.
- Forward the scan-manifest metadata columns into `metadata.json` so they reach reports/BEAM.
- Emit nothing fitted — normalization happens at training time.
