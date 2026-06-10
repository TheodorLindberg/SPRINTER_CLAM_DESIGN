# Specification

The **what-exactly** layer: precise, implementation-independent contracts. Where the [overview](../design/01-overview.md) explains intent, the specification pins down the exact artifact schemas, config field semantics, **invariants**, and **acceptance criteria** that any implementation must satisfy. The [implementation](../impl/index.md) layer then says *how*.

A from-scratch rebuild should be able to target these pages and a test suite should be able to check against them — without reading the implementation.

## Pages

- [WSI Transformation](wsi-transformation.md) — registration outputs, outlines, the biopsy PCA axis.
- [Dataset Preprocessing](preprocessing.md) — coords, embeddings, cache, bundle contracts.
- [Model Training](training.md) — folds, run records, metrics.

*(Being built out stage by stage; other stages follow the same pattern.)*

## How to read a spec page

Each page lists, per artifact: its **schema** (fields, types, layout), the **invariants** that must always hold, and **acceptance criteria** phrased as checks. File-format details defer to the [format specs](../formats/beam.md).
