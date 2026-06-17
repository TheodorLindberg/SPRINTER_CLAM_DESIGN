# 10 · Embedding cache — drop the content-addressed store?

A focused, specific review of one subsystem: the **content-addressed embedding cache** and how it
fights Snakemake and the directory layout. The working hypothesis (yours) is that it should be
dropped for something simpler. This page argues that hypothesis is **right for the usecase**, names
exactly what the cache costs, what it actually buys, and what to replace it with — concretely.

Companion to [09 · Simplification analysis](09-simplification-analysis.md) (deep dive 2), expanded
with the Snakemake/layout specifics.

## What the cache is today

From [`formats/embeddings-and-patches.md`](../formats/embeddings-and-patches.md#content-addressed-cache-key)
and [04](04-snakemake-integration.md#embedding-cache-vs-the-dag):

- Every embedding **row** is keyed by `sha1(round(coords) ∥ patch_size ∥ mpp ∥ model ∥ variant ∥
  aug)` and lives in a **separate per-position store**.
- The per-`(scan, patch_config)` HDF5 that Snakemake actually tracks is an **assembled view** —
  built from cache hits + freshly embedded misses.
- On an overlap change, `embed` does **delta-fill**: diff requested coords vs cached, embed only
  the new positions, copy reused rows by `(x, y)`, and reassemble in a **canonical row order**.

The whole apparatus exists for one purpose: **sub-file reuse** — when two patch grids over the same
scan share positions (e.g. 50%-overlap ⊃ 0%-overlap), don't re-embed the shared ones.

## The core problem: Snakemake is file-grained, the cache is row-grained

Every painful thing about this design flows from one impedance mismatch. Snakemake's DAG reasons
about **files** (existence + mtime); the cache reasons about **rows**. To make a row-level store
coexist with a file-level scheduler, the plan bolts on compensating machinery, and each piece is a
cost or a hazard:

| # | Mechanism (why it exists) | Specific cost / hazard |
|---|---|---|
| 1 | **Canonical row order** — reused + freshly-embedded rows are stitched together, so the assembled file's order must be fixed | A *silent-correctness* hazard: BEAM requires `patches/coords` in the **same order** the model was fed ([eval invariant](../spec/evaluation.md#invariants)). Any reassembly path that orders rows differently misaligns **attention ↔ coordinates** — and nothing crashes. This is the worst kind of bug: invisible, in the scientific output. |
| 2 | **Reuse-by-`(x, y)`** + `round(coords)` in the key | Reuse hinges on exact coordinate equality. A shifted grid origin, an edge-handling tweak, or a rounding difference makes matches silently miss → either re-embed everything (cache defeated) or partial fills. The design itself notes fixed-grid reuse "breaks under outline cropping, edge handling, or a shifted grid origin" — so it chose per-row addressing, which simply *moves* that fragility into `(x,y)`+rounding matching. |
| 3 | **Coords-freshness coupling** — tie `embed`'s output freshness to the coords file so the DAG reruns when `patch_config` changes ([04](04-snakemake-integration.md#embedding-cache-vs-the-dag)) | Pure accidental complexity: a workaround to make a row-level cache *look* consistent to a file-level DAG. It's a rule-wiring subtlety every maintainer must understand to avoid stale or needlessly-recomputed embeddings. |
| 4 | **A parallel store** beside the DAG-tracked files | Two sources of truth to keep consistent; **no eviction policy** is specified, so the per-position store grows **unbounded** across every experiment. Operational cost (disk, cleanup) with no owner. |
| 5 | Cache key must be computed **identically** in the writer and the freshness checker | A correctness coupling across modules ([`shared.hashing`](03-shared-core.md#provenance-hashing)); drift between the two re-embeds or mis-reuses. |

None of these protect a scientific result. Hazard #1 actively *threatens* one.

## What it actually buys — and whether the usecase triggers it

Walk the axes of the cache key and ask which ones vary in practice:

| Key axis | Does it vary? | Is sub-file reuse needed? |
|---|---|---|
| `model` | Yes (conch / uni2_h / …) | **No reuse possible** — different encoder ⇒ different vector. Different files by definition. |
| `source_variant` | Training & metrics are **raw only** | rigid/elastic embeddings are **likely never computed** — heatmaps reuse raw attention warped into the variant frame, they don't re-embed. So this axis is mostly **dead weight**. |
| `aug` | Off, if augmentation is deferred ([09](09-simplification-analysis.md)) | Moot until augmentation proves uplift. |
| `patch_size` / `mpp` / `overlap` | Draft config: **one** size, **`overlap: 0.0`** | This is the *only* axis where sub-file reuse helps — and it currently doesn't vary. |
| `cohort` | Yes (pooled sources) | **Already free** — cohort is *not* in the embedding path; a scan embedded once is reused across every cohort/bundle by plain file existence. The cache's cross-cohort benefit is captured without it. |

So the cache's **unique** benefit over plain files is sub-file reuse across overlap/patch-size — the
one axis the usecase doesn't exercise. Everything else (cross-cohort, cross-bundle reuse) a
file-level scheme already gives for free. **You are paying for a parallel row store and a
silent-misalignment hazard to optimize a sweep you don't run.**

## The simpler solution: let Snakemake be the cache

Drop the per-position store. Make the **embedding file path encode the full config**, and let the
DAG's file existence + timestamps *be* the cache. This is the design's own "make reuse visible to
the DAG itself" idea taken to its honest conclusion.

```text
processed/embeddings/{model}/{aug}/{scan}__{variant}__{patch_config}.h5
```

- **The path is the key.** Same `(scan, variant, model, patch_config, aug)` ⇒ same path ⇒ the file
  already exists ⇒ Snakemake skips the rule. Content-addressing degrades gracefully to "the path
  components are the key."
- **Changing a config recomputes only that target.** A new `patch_config` is a new filename; the
  old one is untouched, the new one is built once. No delta-fill, no reassembly.
- **Rows are written once, in coordinate order, by the embed step.** No stitching ⇒ **hazard #1
  disappears entirely** — `patches/coords` order is whatever `patch_coords` emitted, and BEAM reads
  the same file. Attention/coordinate alignment is structural, not maintained by hand.
- **Cross-cohort/bundle reuse is free** (cohort isn't in the path); bundles **symlink the
  DAG-tracked files directly** ([bundle assembly](05-stage-implementations.md#stage-3-dataset-preprocessing-histomilpreprocessing)) — the parallel store and its symlink-into-cache indirection both vanish.
- **No eviction problem.** There is exactly one embedding file per config you actually ran; delete
  a `patch_config` you abandoned and it's gone. Lifecycle = the DAG.

### Optional, zero-code bonus: Snakemake output caching

Snakemake's built-in `--cache` / output caching keys a rule's output by the hash of its inputs +
code. Mark `embed` cacheable and the **same scan embedded in two workflows/runs** is a cache hit
with **no custom code at all** — covering the cross-run dedup case the content store was partly for.
Whole-output, not sub-file, but that's exactly the granularity that matches the usecase.

## If overlap sweeps ever do matter: decimate, don't delta-fill

Should you later genuinely sweep overlap/patch size on the same scans, adopt the design's
**rejected-but-simpler** fallback rather than resurrecting the row store:

- **Embed once per scan at the finest grid you'll need; coarser grids are a pure-CPU row subset.**
- Robust *by construction*: you define one grid per scan and decimate from it, so a coarser grid
  **is** a subset — none of the cross-config origin/rounding fragility (hazard #2) that made per-row
  addressing seem necessary in the first place.
- Still **no parallel store, no delta-fill, no canonical-order stitching** — subsetting preserves
  order trivially.

That is: the trigger for *any* reuse machinery is "I sweep grids often," and even then the answer is
decimation, not content-addressing.

## Snakemake & layout impact, concretely

What changes if you drop the cache:

- **`preprocess/cache.py` is deleted.** The `embed` rule writes the per-config file directly.
- **`paths.cache_object(key)` is deleted**; `paths.embeddings(...)` (path encodes the full config)
  is the only embedding path authority.
- **The `embed` rule loses the coords-freshness workaround** — it's a normal rule: `input: coords +
  wsi`, `output: the per-config h5`. Standard Snakemake, no special freshness reasoning.
- **`shared.hashing.cache_key` is no longer load-bearing** for embeddings (it may still serve other
  hashes). One fewer cross-module correctness coupling.
- **One directory disappears** (the per-position store) and with it its unbounded-growth/eviction
  question.
- **Doc 04's entire "Embedding cache vs. the DAG" section becomes unnecessary** — a good sign: when
  removing a feature also removes a page of caveats, the feature was net-negative.

## Decision

| | Content-addressed row store (current) | File-level "DAG is the cache" (recommended) | Fine-grid + decimate (only if sweeping) |
|---|---|---|---|
| Sub-file reuse across overlaps | Yes | No | Yes (CPU subset) |
| Cross-cohort / cross-run reuse | Yes | Yes (free) / `--cache` | Yes |
| Canonical-order hazard | **Present** | **None** | None |
| Origin/rounding fragility | Present (in `(x,y)` match) | None | None (one grid per scan) |
| Parallel store + eviction | Yes | No | No |
| Custom code | High | ~None | Low (pure-CPU subset) |
| Triggers in this usecase | the unused axis | matches it | only if grid sweeps appear |

**Recommendation: drop the content-addressed store now.** Use file-level caching where the path
encodes the full config; optionally enable Snakemake output caching for cross-run dedup. Revisit
*only* if a real experiment plan sweeps overlap/patch size on shared scans — and then reach for
fine-grid decimation, not the row store. This removes a silent-correctness hazard from the
scientific output, deletes a module and a directory, and erases a page of Snakemake caveats, at the
cost of re-embedding a sweep you don't currently run.

> One thing to confirm: that you really only ever embed `raw`. If a future experiment trains on
> registered embeddings, the `variant` axis comes alive — but it's still just another path
> component in the file-level scheme, not a reason to bring back the row store.
