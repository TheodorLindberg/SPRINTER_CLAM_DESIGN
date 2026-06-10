# Configuration

The Snakemake-driven stages each take a YAML config. Since stages run individually (and increasingly often), each has its own config file rather than one monolith. Stage 1 (Data Ingestion) is outside Snakemake and user-written, so it has no config here.

!!! warning "These are drafts"
    The config files in `configs/` are early drafts to anchor discussion — field names and structure will change as stages are implemented.

## Config files

| Config | Stage | Controls |
|---|---|---|
| `configs/wsi_transformation.yaml` | 2 | Variants to produce, registration backend, outline + quartile options |
| `configs/preprocessing.yaml` | 3 | Derived labels, patching, embedding model, bundle + **patient exclusion** |
| `configs/training.yaml` | 4 | Target label, architecture, hyperparameters, **balancing**, **augmentation**, sweep, HPO |
| `configs/evaluation.yaml` | 5 | Bundle + model to evaluate, attention variants, output |
| `configs/heatmaps.yaml` | 6 | Source variant, attention type, colormap, render + output options |
| `configs/seeds.yaml` | 4 | Split registry — dataset, fold/model seeds, fold count (shared by all models) |

## Cross-cutting notes

- **`seeds.yaml` is the split registry.** All models reference it so fold splits are identical and seed-addressable by name. A change in patient count raises a stale-splits warning.
- **Patient exclusion** lives in `preprocessing.yaml` and produces the three bundles (training / full / held-out).
- **Balancing and augmentation** live in `training.yaml` and apply per fold from the training split only.
- A single merged `config.yaml` with per-stage sections is a possible alternative if the per-file split proves cumbersome — left open.
