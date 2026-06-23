# Embeddings & patches

Patch coordinates and patch embeddings are stored as binary **HDF5**, per scan. Pixel crops are *not* stored — pixels are read from the WSI on demand at embedding time. Produced in [Stage 3 · Dataset Preprocessing](../design/05-dataset-preprocessing.md).

---

## Per-scan embedding file

```text
{scan_id}__src-{source_variant}__emb-{embedding_model_id}__patch-{patch_config_id}.h5
│
├─ ⚙ attributes
│     scan_id, source_variant
│     embedding_model_id
│     patch_config_id, patch_size, patch_resolution
│     embedding_dim
│
├─ coords ················· (N, 2|4) int     x, y (, w, h) · WSI frame
├─ embeddings ············· (N, D) float    RAW model output — never altered
├─ quartile ··············· optional (N,) int8     biopsy-axis region, forwarded from the coords file
├─ axis_t ················· optional (N,) float32  normalized arc-length position [0,1]
└─ axis_offset ············ optional (N,) float32   signed lateral distance from the axis curve
```

`quartile`/`axis_t`/`axis_offset` are forwarded verbatim from the patch-coords file (see
[WSI Transformation spec](../spec/wsi-transformation.md#biopsy-axis-skeleton-curve)) — absent
entirely on a file written before they existed, so older cache entries stay valid. They never
change the `embeddings` array itself: appending `axis_t`/`axis_offset` onto the feature vector a
model trains on is a **training-time** decision (`append_axis_features`, see the
[training spec](../spec/training.md)), made once when a bundle's bags are loaded — not baked into
this cache, so the same cached embeddings serve both a run that uses axis features and one that
doesn't.

---

## Cache identity (the file path)

There is no separate cache store: the **file path is the key**. One HDF5 holds the embeddings for one `(scan, source_variant, embedding_model, patch_config)`, and everything that affects an embedding's value is captured in that path:

| Component | Why it identifies the file |
|---|---|
| `scan_id` + `source_variant` | which pixels (raw / rigid / elastic differ) |
| `patch_config` | patch size, resolution, and overlap — the crop extent and grid |
| `embedding_model_id` | different encoder → different vector |

Snakemake tracks the file directly, so an already-embedded configuration is skipped and a changed one writes a new file; rows are written once in coordinate order. Reuse across cohorts is automatic because the cohort is not part of the path. See [the reuse decision](../design/09-open-questions.md#embedding-reuse-strategy).

---

## In the bundle

The bundle symlinks or copies these per-scan HDF5 files. Patch geometry can additionally be exported as GeoJSON for TissUUmaps viewing — see [Outlines & geometry exports](outlines.md#geojson-export).
