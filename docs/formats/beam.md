# BEAM format

**BEAM** — *Biopsy Evaluation & Attention Map* — is the project's own per-biopsy, per-sweep evaluation result, stored as **HDF5**:

```text
{biopsy_id}__{run_family}.beam.h5
```

One BEAM covers a whole **seed sweep** — every `fold_seed × model_seed` model fanned out from one experiment-config run entry (`run_family`, e.g. `"ki67_conch"`) — not one individual model. Every contributing model's own prediction, attention, and full stats live side by side under its own `models/{run_id}/` group, so a model's performance is readable from the BEAM alone without cross-referencing its `run.json`.

How it is generated (checkpoint routing, per-model aggregation, sweep invariants) is the [Evaluation spec](../spec/evaluation.md).

Produced by [Stage 5 · Evaluation](../design/07-evaluation.md); consumed by reports and [Stage 6 · Heatmaps](../design/08-heatmaps.md). HDF5 is chosen because it is **appendable** — enrichment steps can add datasets or groups without breaking existing readers.

!!! tip "Provisional name"
    The format name can be renamed with a single find-replace.

---

## Layout

```text
{biopsy_id}__{run_family}.beam.h5
│
├─ ⚙ attributes ··········· provenance, shared across every contributing model — see table below
│
├─ patches/ ················ shared — one bag, same patches fed to every model
│  ├─ coords ·············· (N, 2|4) int   x, y (, w, h) · WSI frame
│  └─ size ················ patch size / resolution
│
├─ outline/ ················ shared — polygon arrays, identical to the Outlines spec
│  └─ components/ ········· optional · one (Mᵢ, 2) float array per tissue island
│
├─ labels/ ·················· shared — true labels where available · name → value (+ type)
├─ embeddings ··············· shared, optional · (N, D) float · else referenced from bundle
├─ metadata/ ················ shared — model + registration info, free-form, plus
│  ├─ dataset/ ··········· forwarded scan-manifest metadata, grouped by level
│  ├─ patient/             (each value keeps the level it was defined at)
│  ├─ biopsy/
│  └─ scan/
│
└─ models/ ·················· one group PER CONTRIBUTING MODEL — the sweep's fan-in
   └─ {run_id}/ ············ e.g. "ki67_conch__fs0__ms200"
      ├─ ⚙ fold_seed, model_seed, checkpoints_used, git_commit
      ├─ ⚙ architecture_json, hyperparameters_json,
      │    target_normalization_json, metrics_json    # JSON-encoded — a full
      │                                                # copy of that model's
      │                                                # RunRecord fields
      ├─ prediction ·········· that model's bag-level prediction
      └─ attention/ ·········· ONLY for that model's architecture if attention-based (CLAM);
         ├─ raw ·············· (N,) float                omitted otherwise — a sweep may
         ├─ sigmoid ·········· (N,) float                freely mix attention and
         └─ rank ············· (N,) float                non-attention models
```

---

## Root attributes

Shared across every contributing model in the sweep (one bag, one biopsy):

| Attribute | Meaning |
|---|---|
| `format_version` | BEAM schema version |
| `biopsy_id`, `patient_id`, `dataset_id` | Entity provenance |
| `run_family`, `model_experiment` | Which sweep (experiment-config run entry) + its experiment umbrella |
| `model_ids` | Every contributing model's `run_id`, computed by the writer from what was actually written (never hand-supplied) |
| `bundle_id`, `cohort_id` | The bundle scored and its cohort — identical for every model in the sweep |
| `embedding_model_id` | Embedder that produced the bag |
| `subset` | `development` / `holdout` / `all` — identical for every model in the sweep |
| `evaluation_tag` | Readable evaluation-run name |
| `membership_hash`, `git_commit` | Reproducibility state — `membership_hash` (cohort membership, identical across the sweep) and `git_commit` (the **evaluation** run's own code state, when this BEAM was generated — distinct from each model's own training-time `git_commit` under `models/{run_id}/`) |
| `stain` | Stain of the evaluated scan(s) |
| `source_variant` | `raw` / `rigid` / `elastic` |
| `patch_config_id`, `patch_size`, `patch_resolution` | Patching configuration |
| `quartile` | Carried metadata — **not** a spatial index |

## Per-model attributes (`models/{run_id}/`)

A full copy of that model's contributing `RunRecord` — every field a consumer needs to know exactly how this one model in the sweep did, without re-opening its `run.json`. Nested dicts (`architecture`, `hyperparameters`, `target_normalization`, `metrics`) are JSON-encoded into `*_json` string attrs (HDF5 attrs cannot hold arbitrary nested dicts):

| Attribute | Meaning |
|---|---|
| `fold_seed`, `model_seed` | This model's seed tags |
| `checkpoints_used` | Which of this model's fold checkpoints contributed (one for development/all, every fold for holdout) |
| `architecture_json`, `hyperparameters_json` | This model's architecture + hyperparameters, verbatim from its `RunRecord` |
| `target_normalization_json`, `metrics_json` | This model's normalization stats + train/val/test metrics, verbatim from its `RunRecord` |
| `git_commit` | This model's own training-time code state |

---

## Outline

The tissue outline used follows the **exact same layout as the [Outlines spec](outlines.md#polygon-array)** — one polygon vertex array per kept tissue component (`outline/components/0, 1, …`, largest first). GeoJSON is produced only as a separate viewing export, never stored inside BEAM. Shared across every model (the same biopsy).

The per-patch region/position (`quartile`/`axis_t`/`axis_offset` — see [Embeddings & patches](embeddings-and-patches.md)) are not currently carried into BEAM's `patches/`: they flow from the coords file into the embeddings file, and BEAM's `patches/coords` is sourced from the embeddings file, which doesn't yet forward them. Wiring that through is deferred alongside the future MIL-architecture work that would actually consume them.

---

## Field mapping

| Evaluation field | Location | Scope |
|---|---|---|
| Attention (raw, sigmoid, rank) | `models/{run_id}/attention/{raw,sigmoid,rank}` — attention models only | per model |
| Prediction | `models/{run_id}/prediction` | per model |
| Model stats (architecture, hyperparameters, normalization, metrics) | `models/{run_id}/@*_json` | per model |
| True labels (where available) | `labels/` | shared |
| Patches | `patches/coords` | shared |
| Patch size | attr `patch_size` + `patches/size` | shared |
| Tissue outline used | `outline/components/` (one polygon array per island) | shared |
| Whole-scan-vs-quartile-bag marker (carried label metadata, not geometry — see root attrs) | attr `quartile` | shared |
| Stain | attr `stain` | shared |
| Patient | attr `patient_id` | shared |
| Sweep + contributing models | attr `run_family`, `model_ids` + `metadata/` | shared |
| Embedding model | attr `embedding_model_id` | shared |
| Registration / source variant | attr `source_variant` + `metadata/` | shared |
| Metadata | `metadata/` | shared |
