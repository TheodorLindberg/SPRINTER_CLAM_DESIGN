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

## Normalized format: folders vs. flat {#normalized-format-folders-vs-flat}

**Recommended default: support both; make the manifest the contract.**

Rather than picking one layout as canonical, the pipeline should **not depend on the on-disk layout at all**. Ingestion emits a manifest of IDs → file paths, and every downstream stage reads that. The folder hierarchy (`patient_<x>/biopsy_<x>/scan_<stain>`) becomes the *suggested* human-facing convention, while a flat naming scheme remains valid because only the manifest is load-bearing.

This dissolves the either/or and keeps user-written ingesters flexible.

**Still open:** the exact manifest schema (the pre-preprocessing structure).

---

## Patient-exclusion leakage {#patient-exclusion-leakage}

**Recommended default: no fitted statistics in bundles.**

The leakage risk in the three-bundle scheme (training / full / held-out) is *derived state* — chiefly label normalization statistics, but also any quantity fitted on the cohort (distribution-derived thresholds, class weights).

The clean rule: bundles carry **raw** labels and embeddings only. Anything fitted is computed at training time from the **training split alone**. The three bundles then differ only in which raw rows they contain and share no derived state by construction — converting the validation worry into a design invariant.

**Still open:** confirm that any embedding-time stain normalization is a fixed pretrained transform (not fitted on this cohort); if it were fitted, it would need the same treatment.

---

## Metadata file scope {#metadata-file-scope}

**Recommended default: extra columns on the existing entity manifests; no separate metadata file yet.**

Patient, biopsy, and scan form a strict hierarchy, so metadata at a coarser level is always reachable at a finer level by ID inheritance (a scan inherits its biopsy's and patient's metadata via the join). That means we never *need* three separate metadata stores — one mechanism plus the hierarchy suffices.

The minimal version: allow arbitrary extra columns on the per-entity manifests (patients / biopsies / scans) and let downstream stages join by ID. This supports all three granularities with no new machinery.

**Still open:** whether a free-form JSON blob column is needed for deeply nested or irregular metadata. Flat columns now; JSON blob deferred until a real case demands it.

---

## Genuinely open

- **Combining input data sources in CLAM training preparation.** How to merge different input data sources when assembling CLAM training inputs is undecided.
- **Exact bundle schema.** The schema shared by training and evaluation must be defined before implementation; deferred while the stage contracts settle.
