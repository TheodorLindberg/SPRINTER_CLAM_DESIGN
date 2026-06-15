# Open Questions & Decisions

Working notes for discussion with the team. Items marked **Recommended default** have a proposed resolution with rationale; items marked **Open** are genuinely undecided.

---

## Embedding reuse strategy

**Recommended default: content-addressed cache.**

Three options were considered:

| Approach | How | Verdict |
|---|---|---|
| **Content-addressed cache** | Key each embedding by coordinates + patch size + resolution + embedding model + source variant; embed only cache misses. | **Chosen.** Robust across the full overlap × stain × model × variant combinatorics; reuse survives outline cropping, edge handling, and shifted grid origins. |
| Canonical fine grid + decimation | Embed once at the finest overlap ever needed; derive coarser sets by subsetting. | Reasonable fallback if we'd rather avoid a cache layer, but pays the full dense cost upfront and fixes the finest grid in advance. |
| Fixed global grid origin | Force all grids for a scan to share one origin/step so coarser grids are subsets by construction. | Cheapest, but brittle — anything perturbing the origin silently invalidates reuse. |

!!! tip "Refinement"
    Include **patch resolution / pyramid level** in the cache key, not just coordinates and size — the same coordinates at a different magnification give different pixel content and must not collide.

**Still open:** whether to invest in the cache layer now or start with decimation and migrate later.

---

## Holdout leakage {#patient-exclusion-leakage}

**Recommended default: no fitted statistics in bundles + holdout filtered from all folds.**

The leakage risk is *derived state* — chiefly label normalization statistics, but also any quantity fitted on the data (distribution-derived thresholds, class weights).

The clean rule: bundles carry **raw** labels and embeddings only. Anything fitted is computed at training time from the **training fold alone**. Combined with `holdout` patients being filtered out of every fold (they are only ever scored at evaluation), holdout shares no derived state with development by construction — converting the validation worry into a design invariant. See [Cohorts, roles, and splits](02-data-model.md#cohorts-roles-and-splits).

**Still open:** confirm that any embedding-time stain normalization is a fixed pretrained transform (not fitted on this cohort); if it were fitted, it would need the same treatment.

---

## Metadata file scope {#metadata-file-scope}

**Decided: nested `metadata:` per level in the YAML/JSON scan manifest, forwarded into the bundle and BEAM grouped by level.**

The scan manifest is hierarchical (`dataset → patient → biopsy → scan`), so metadata sits as a `metadata:` block at whichever level it describes — written once, no repetition. Preprocessing forwards it downstream **keeping the level tag**, and the [BEAM](../formats/beam.md) `/metadata` stores it grouped `dataset/ patient/ biopsy/ scan/`, so a reader always knows what a value describes. No separate metadata store, and the nested form handles irregular/structured metadata directly. See [Data Ingestion · Scan manifest](03-data-ingestion.md#the-scan-manifest).

---

## Stains per bundle

**Decided: one stain per bundle (one model per stain) for now.** A biopsy's HE / Ki67 / PSA scans become separate bundles, each feeding its own model, so evaluation's per-bag → per-biopsy step is 1:1.

### Future: multi-stain, multi-target models

Example: a single model that takes **both Ki67 and PSA** patch embeddings for a biopsy and predicts **both** scores. To support this, the current contracts extend rather than change (single-stain stays the one-element case):

- **Bundle** — allow `stains: [Ki67, PSA]`; the bundle then carries one bag per `(biopsy, stain)`, already distinguishable via `bag_id`'s stain field.
- **Architecture** — one MIL branch per stain (each pools its own bag), fused at a head with **per-target outputs** (multi-task: one head per score). No spatial alignment needed — each stain is pooled independently and fused at the prediction level — so it stays compatible with raw-frame training.
- **Config** — `target` becomes a set (e.g. `{ki67_score, psa_score}`); balancing and metrics are computed per target.
- **Evaluation / BEAM** — this is where the per-bag → per-biopsy step becomes a **real aggregation**: attention stored per stain (`attention/Ki67`, `attention/PSA`), predictions per target.

## Genuinely open

- **Holdout ensembling policy.** Holdout patients are scored by all fold checkpoints; default is the **mean** prediction. Confirm vs. alternatives (median, best-fold, per-checkpoint kept).
- **Combining input data sources in CLAM training preparation.** How to merge different input data sources when assembling CLAM training inputs is undecided.
- **Exact bundle schema.** The schema shared by training and evaluation must be defined before implementation; deferred while the stage contracts settle.
