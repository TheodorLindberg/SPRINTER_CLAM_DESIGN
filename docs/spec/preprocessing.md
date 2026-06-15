# Spec · Dataset Preprocessing

Contracts for [Stage 3](../design/05-dataset-preprocessing.md). [Overview](../design/05-dataset-preprocessing.md) · **Specification** · [Implementation](../impl/preprocessing.md).

## Cohort resolution (per cohort)

`resolve_cohort` turns a [`cohorts.yaml`](../configs/cohorts.md) entry into a frozen, validated membership before any bundle or fold is built.

**`membership.csv`** — `dataset_id, patient_id, biopsy_id, role` (`role ∈ {development, holdout}`), plus a recorded **`membership_hash`** over the sorted rows.

### Validation (hard errors)

- Every listed patient exists in its dataset's [scan manifest](../design/03-data-ingestion.md#the-scan-manifest).
- Every `holdout` patient is a member; no patient has more than one role.
- A fraction+seed holdout is reproducible (same members + seed → same split).
- Across pooled datasets, the target label's `name`/`type` are comparable (no silent scale mismatch).

### Warnings (logged, not fatal)

- A member patient missing a requested stain (it simply contributes no bag).
- A development patient with no value for the experiment target.

### Report

`results/reports/cohorts/{cohort}.html` — composition (members by dataset × role), label distribution per role, and the `membership_hash` + provenance. Built with the [report toolkit](../design/11-reports.md) (Plotly + standalone CSS), emitted here so a cohort can be eyeballed **before** heavy preprocessing.

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

Per-row addressable; a run embeds only keys absent from the cache. The cache is a **separate per-position store**; the per-`patch_config` embedding file (which Snakemake tracks) is assembled from cache hits + freshly embedded misses.

## Bundle (a prepared cohort)

```text
{bundle_id}/
  manifest.csv        # one row per bag
  labels.csv          # all labels (absent if label-free)
  metadata.json       # cohort_id, membership_hash, embedding_model_id, patch_config, source_variant, forwarded scan-manifest metadata
  embeddings/{bag_id}.h5 -> symlink
  outlines/{scan_id}.geojson -> symlink
```

**manifest.csv** columns: `bag_id, dataset_id, patient_id, biopsy_id, stain, role, embedding_path, n_patches`.
**labels.csv** columns: `dataset_id, patient_id, biopsy_id, label_name, label_value, label_type`.

!!! warning "Multi-dataset key"
    `biopsy_id` is unique only within a patient, so labels and bags join on the **full `(dataset_id, patient_id, biopsy_id)`** key — never `biopsy_id` alone — or a pooled cohort would mismatch rows.

## Invariants

- Every `manifest` row's `embedding_path` exists and its `n_patches` equals the embedding row count.
- `role ∈ {development, holdout}`; no fold information is present (folds belong to training).
- `metadata.membership_hash` matches the cohort's frozen membership.
- **No fitted statistics** anywhere in the bundle (raw labels and embeddings only).
- A label-free bundle omits `labels.csv` and downstream stages tolerate its absence.
- Augmented embedding sets (if enabled) appear as extra bags tagged `augmented`, used in **training folds only** — never `val` / `test` / `holdout`, and always in their patient's fold (no leakage, no optimistic metrics).

## Acceptance criteria

- Rebuilding a bundle from the same cohort + cache is a no-op (identical manifest + same symlink targets).
- Changing `overlap` only adds new patch coords; shared positions reuse cached embeddings (no re-embedding).
- Every `development` and `holdout` patient of the cohort that has the required stain appears; patients lacking the stain are absent (logged), not errors.
