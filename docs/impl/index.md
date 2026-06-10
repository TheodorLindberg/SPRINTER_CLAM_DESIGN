# Implementation

The **how-to** layer: the build recipe behind each [specification](../spec/index.md). Algorithms (with the math where it matters), function/CLI signatures, recommended libraries, pseudocode, and edge-case handling.

This layer can change without changing the contract — as long as the [spec](../spec/index.md)'s schemas, invariants, and acceptance criteria still hold.

## Pages

- [WSI Transformation](wsi-transformation.md) — VALIS registration, mask + outline extraction, the PCA biopsy-axis algorithm.
- [Dataset Preprocessing](preprocessing.md) — patching, embedding, the content-addressed cache, bundle assembly.
- [Model Training](training.md) — fold generation, the training loop, balancing, HPO.

*(Being built out stage by stage.)*

## Conventions

- Pseudocode is illustrative, not prescriptive of language; Python is assumed for the reference build.
- Recommended libraries are named where one is clearly suited (VALIS, OpenCV, scikit-image, shapely, zarr/tifffile, PyTorch/timm, Optuna), but are not part of the contract.
