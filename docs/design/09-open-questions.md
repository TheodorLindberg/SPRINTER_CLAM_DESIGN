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

**Decided: metadata rides on the scan manifest as columns at any entity level, and is forwarded through preprocessing into the bundle and BEAM.**

Patient, biopsy, and scan form a strict hierarchy, so metadata at a coarser level is always reachable at a finer level (patient-level columns repeat across that patient's rows). No separate metadata store is needed — extra manifest columns plus the hierarchy suffice. Preprocessing carries this metadata forward so it is present in the bundle manifest and in each BEAM file, without re-joining to the source. See [Data Ingestion · Scan manifest](03-data-ingestion.md#scan-manifest-the-contract).

**Still open:** whether a free-form JSON blob column is needed for deeply nested or irregular metadata. Flat columns now; JSON blob deferred until a real case demands it.

---

## Genuinely open

- **Single- vs multi-stain bundles.** A bundle's `stains` is a list, but the reference model (CLAM) is single-stain. Is a bundle always one stain (one model per stain — current assumption), or can one model consume several stains' bags per biopsy? This decides whether evaluation's per-bag → per-biopsy step is 1:1 or a real aggregation. Default: **single-stain** until a multi-stain architecture exists.
- **Holdout ensembling policy.** Holdout patients are scored by all fold checkpoints; default is the **mean** prediction. Confirm vs. alternatives (median, best-fold, per-checkpoint kept).
- **Combining input data sources in CLAM training preparation.** How to merge different input data sources when assembling CLAM training inputs is undecided.
- **Exact bundle schema.** The schema shared by training and evaluation must be defined before implementation; deferred while the stage contracts settle.
