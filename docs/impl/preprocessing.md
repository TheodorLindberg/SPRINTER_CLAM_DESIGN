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
model = load_model(embedding_model_id)            # registry: resnet*, uni2_h, conch, gigapath
slide = open_zarr(wsi_path)                        # tifffile→zarr; no full decompress
for batch in batched(coords):
    imgs = [read_patch(slide, x, y, patch_size, level) for x, y in batch]
    imgs = normalize(imgs, model.NORM)             # model's own mean/std (not cohort-fitted)
    emb  = model(to_tensor(imgs))                  # (B, D)
    append(emb)
save_h5(coords, embeddings, attrs)
```

- Device: respect `CUDA_VISIBLE_DEVICES`; else pick the GPU with most free memory.
- HuggingFace offline / SSL workarounds as needed on HPC.

### Content-addressed cache

Compute the [key](../spec/preprocessing.md#cache-key) per scan-config; diff requested vs cached coords; **embed only the delta**, copy reused rows by `(x,y)` index, reassemble in requested order. (Generalises the old `coords_hash` incremental re-embed.)

### Augmentation

For each configured transform set (flip / rotate90 / stain-jitter in HED space), apply to the patch image **before** embedding and cache under a distinct `augmentation_id`. `n_variants` controls how many augmented copies per patch. The foundation model runs here, never in the training loop.

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
