# Stage 4 · `training.yaml`

An **experiment definition**: one file that fans out into many runs (seed sweep × optional HPO) — no per-run files. It names the umbrella `experiment`, the `bundles` it spans (possibly multiple stains/embeddings), the **cohort scope** (`development` for CV, `all` for final retrain), the `seed_set`, architecture, balancing, and an `hpo` block. HPO models are **segregated** under `results/experiments/{exp}/hpo/` with their own index; you keep the best and run a seed sweep on them. See [Stage 4 · Model Training](../design/06-model-training.md) and [Reports](../design/11-reports.md).

```yaml title="training.yaml"
--8<-- "training.yaml"
```
