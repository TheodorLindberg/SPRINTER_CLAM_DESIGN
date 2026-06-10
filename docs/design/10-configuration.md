# Configuration

The Snakemake-driven stages each take a YAML config. Since stages run individually (and increasingly often), each has its own config file rather than one monolith. Stage 1 (Data Ingestion) is outside Snakemake and user-written, so it has no config here.

!!! warning "These are drafts"
    The configs below live in `docs/configs/` and are embedded here directly — the YAML you see *is* the file. Field names and structure will change as stages are implemented.

## Cross-cutting notes

- **`seeds.yaml` is the split registry.** All models reference it so fold splits are identical and seed-addressable by name. A change in patient count raises a stale-splits warning.
- **Patient exclusion** lives in `preprocessing.yaml` and produces the three bundles (training / full / held-out).
- **Balancing and augmentation** live in `training.yaml` and apply per fold from the training split only.
- A single merged `config.yaml` with per-stage sections is a possible alternative if the per-file split proves cumbersome — left open.

---

## Split registry — `seeds.yaml`

Names the dataset and the seeds every model shares, so fold splits are reproducible and referenceable by name. Used by Stage 4.

```yaml title="seeds.yaml"
--8<-- "seeds.yaml"
```

---

## Stage 2 · WSI Transformation — `wsi_transformation.yaml`

Which source variants to produce, the registration backend, and outline options (including geometric quartile subdivision and the GeoJSON export for TissUUmaps).

```yaml title="wsi_transformation.yaml"
--8<-- "wsi_transformation.yaml"
```

---

## Stage 3 · Dataset Preprocessing — `preprocessing.yaml`

Derived labels, patching strategy, embedding model + cache, and bundle assembly. The `patient_exclusion` block drives the three-bundle output.

```yaml title="preprocessing.yaml"
--8<-- "preprocessing.yaml"
```

---

## Stage 4 · Model Training — `training.yaml`

Target label, architecture (family → type → params), hyperparameters, **label balancing**, **augmentation**, the seed sweep, and HPO.

```yaml title="training.yaml"
--8<-- "training.yaml"
```

---

## Stage 5 · Evaluation — `evaluation.yaml`

Which bundle and model to evaluate, which attention variants to write into the [BEAM files](../formats/beam.md), and where to put them.

```yaml title="evaluation.yaml"
--8<-- "evaluation.yaml"
```

---

## Stage 6 · Heatmap Generation — `heatmaps.yaml`

Underlay source variant, attention type and colormap, and render/output options. Note the raw-frame coordinate caveat in [Heatmaps](08-heatmaps.md).

```yaml title="heatmaps.yaml"
--8<-- "heatmaps.yaml"
```
