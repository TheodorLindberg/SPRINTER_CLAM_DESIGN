# Implementation

The **how-to** layer: design-level notes on the approach behind each [specification](../spec/index.md) — algorithms (with the math where it matters), recommended libraries, orchestration, and edge-case handling.

This layer can change without changing the contract, as long as the [spec](../spec/index.md)'s schemas, invariants, and acceptance criteria still hold.

!!! warning "No prescriptive code yet"
    These pages describe *how a stage is meant to work*, not a verified build. The earlier pseudocode was reverse-engineered from a previous codebase, so it has been removed to avoid presenting unproven code as authoritative. Concrete code examples will be reintroduced once the real implementation exists and can be referenced. Until then, the [specs](../spec/index.md) are the source of truth for behaviour.

## Pages

- [Snakemake workflow](workflow.md) — every rule, its dependencies, and the config that drives it.
- [WSI Transformation](wsi-transformation.md) — registration, mask + outline extraction, the PCA biopsy-axis approach.
- [Dataset Preprocessing](preprocessing.md) — patching, embedding, the file-level cache, bundle assembly.
- [Model Training](training.md) — fold generation, the training loop, balancing, HPO.
- [Evaluation](evaluation.md) — per-model batched inference and BEAM generation.

*(Being built out stage by stage.)*

## Conventions

- Notes are language-agnostic on intent; Python is assumed for the eventual reference build.
- Recommended libraries are named where one is clearly suited (VALIS, OpenCV, scikit-image, shapely, zarr/tifffile, PyTorch/timm, Optuna), but are not part of the contract.
