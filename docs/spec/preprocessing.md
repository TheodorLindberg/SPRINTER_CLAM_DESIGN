# Spec · Dataset Preprocessing

Contracts for [Stage 3](../design/05-dataset-preprocessing.md). [Overview](../design/05-dataset-preprocessing.md) · **Specification** · [Implementation](../impl/preprocessing.md).

## Patch coordinates (per scan · patch_config)

HDF5, see [Embeddings & patches](../formats/embeddings-and-patches.md). Required:

| Field | Type | Notes |
|---|---|---|
| `coords` | int32 `(N, 2)` | x, y of patch top-left, level-0 frame |
| attr `patch_size` | int | pixels at `level` |
| attr `level` / `mpp` | int / float | pyramid level and microns-per-pixel |
| attr `source_variant` | str | `raw` / `rigid` / `elastic` |
| attr `quartile` | int8 `(N,)` | 1–4 from the [biopsy axis](wsi-transformation.md#biopsy-axis-pca-line); 0 = unassigned |

## Embeddings (per scan · variant · model · augmentation)

| Field | Type | Notes |
|---|---|---|
| `coords` | int32 `(N, 2)` | matches the coord file |
| `embeddings` | float32 `(N, D)` | **raw** model output — no fitted normalization |
| attr `embedding_model_id`, `embedding_dim`, `patch_size`, `mpp`, `source_variant`, `augmentation_id` | | provenance + cache key components |

### Cache key

```
key = sha1( round(coords) ∥ patch_size ∥ mpp ∥ embedding_model_id ∥ source_variant ∥ augmentation_id )
```

Per-row addressable; a run embeds only keys absent from the cache.

## Bundle (a prepared cohort)

```text
{bundle_id}/
  manifest.csv        # one row per bag
  labels.csv          # all labels (absent if label-free)
  metadata.json       # cohort_id, membership_hash, embedding_model_id, patch_config, source_variant, forwarded scan-manifest metadata
  embeddings/{bag_id}.h5 -> symlink
  outlines/{scan_id}.geojson -> symlink
```

**manifest.csv** columns: `bag_id, biopsy_id, patient_id, dataset_id, stain, role, embedding_path, n_patches`.
**labels.csv** columns: `biopsy_id, label_name, label_value, label_type`.

## Invariants

- Every `manifest` row's `embedding_path` exists and its `n_patches` equals the embedding row count.
- `role ∈ {development, holdout}`; no fold information is present (folds belong to training).
- `metadata.membership_hash` matches the cohort's frozen membership.
- **No fitted statistics** anywhere in the bundle (raw labels and embeddings only).
- A label-free bundle omits `labels.csv` and downstream stages tolerate its absence.

## Acceptance criteria

- Rebuilding a bundle from the same cohort + cache is a no-op (identical manifest + same symlink targets).
- Changing `overlap` only adds new patch coords; shared positions reuse cached embeddings (no re-embedding).
- Every `development` and `holdout` patient of the cohort that has the required stain appears; patients lacking the stain are absent (logged), not errors.
