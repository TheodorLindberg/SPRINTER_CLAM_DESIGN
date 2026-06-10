# BEAM format

**BEAM** — *Biopsy Evaluation & Attention Map* — is the project's own per-biopsy, per-model evaluation result format. Stored as **HDF5**, named `{biopsy_id}__{model_id}.beam.h5`.

Produced by [Stage 5 · Evaluation](../design/07-evaluation.md); consumed by reports and [Stage 6 · Heatmaps](../design/08-heatmaps.md).

HDF5 is chosen because it is **appendable** — enrichment steps can add datasets or groups (extra heatmaps, additional embeddings, new attention variants) without breaking existing readers.

> The format name is provisional and can be renamed with a single find-replace.

## Layout

```text
{biopsy_id}__{model_id}.beam.h5
  /                         # root attributes (provenance)
      format_version
      biopsy_id, patient_id, dataset_id
      model_id, embedding_model_id
      evaluation_tag
      stain
      source_variant        # raw / rigid / elastic
      patch_config_id, patch_size, patch_resolution
      quartile              # carried metadata, NOT a spatial index
  /prediction               # prediction per model
  /labels                   # true labels where available (name → value, with type attrs)
  /patches
      coords                # (N, 2|4) int — x, y (, w, h) in the WSI frame, binary
      size                  # patch size / resolution
  /attention
      raw                   # (N,)
      sigmoid               # (N,)
      rank                  # (N,)
  /embeddings               # optional (N, D) — included or referenced from the bundle
  /outline                  # see below — polygon arrays
  /metadata                 # model info, embedding model, registration / source-variant info, free-form
```

## Outline

The tissue outline used is stored as a **polygon vertex array**, not a GeoJSON string (GeoJSON is produced only as a separate viewing export — see [Outlines](outlines.md)).

```text
  /outline
      polygon               # (M, 2) float — outline vertices in the WSI frame
      quartiles/            # optional: outline divided along the biopsy length
          q0                # (M0, 2) float
          q1                # (M1, 2) float
          q2                # (M2, 2) float
          q3                # (M3, 2) float
```

Quartile subdivision is **geometric** (split along the biopsy's long axis), provided so the per-quartile Ki-67 / PSA scores can be associated with approximate regions. True per-region mapping remains unavailable, so this is an approximation, not ground-truth localization.

## Field mapping

| Evaluation field | Location |
|---|---|
| Attention (raw, sigmoid, rank) | `/attention/{raw,sigmoid,rank}` |
| Prediction per model | `/prediction` |
| True labels (where available) | `/labels` |
| Stain | root attr `stain` |
| Metadata | `/metadata` |
| Patch size | root attr + `/patches/size` |
| Model info | `/metadata`, root attr `model_id` |
| Embedding model | root attr `embedding_model_id` |
| Patient | root attr `patient_id` |
| Tissue outline used | `/outline` (polygon array) |
| Patches | `/patches/coords` |
| Quartile | root attr `quartile`; `/outline/quartiles` when subdivided |
| Registration / source variant | root attr `source_variant`, `/metadata` |
