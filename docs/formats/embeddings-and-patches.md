# Embeddings & patches

Patch coordinates and patch embeddings are stored as binary **HDF5**, per scan. Pixel crops are *not* stored — pixels are read from the WSI on demand at embedding time. Produced in [Stage 3 · Dataset Preprocessing](../design/05-dataset-preprocessing.md).

---

## Per-scan embedding file

```text
{scan_id}__src-{source_variant}__emb-{embedding_model_id}.h5
│
├─ ⚙ attributes
│     scan_id, source_variant
│     embedding_model_id
│     patch_size, patch_resolution
│     embedding_dim
│
├─ coords ················· (N, 2|4) int     x, y (, w, h) · WSI frame
└─ embeddings ············· (N, D) float
```

---

## Content-addressed cache key

Each embedding row is addressed by everything that affects its value:

| Component | Why it's in the key |
|---|---|
| `coords` | Different positions → different patches |
| `patch_size` | Different crop extent |
| `patch_resolution` | Same coords at another magnification ≠ same content |
| `embedding_model_id` | Different encoder → different vector |
| `source_variant` | raw / rigid / elastic differ in pixels |
| `augmentation_id` | Augmented patch variants are embedded and cached separately |

A run looks up by key and embeds only cache misses, so different overlap settings reuse shared positions automatically. See [the reuse decision](../design/09-open-questions.md#embedding-reuse-strategy).

---

## In the bundle

The bundle symlinks or copies these per-scan HDF5 files. Patch geometry can additionally be exported as GeoJSON for TissUUmaps viewing — see [Outlines & geometry exports](outlines.md#geojson-export).
