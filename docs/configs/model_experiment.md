# Stage 4 · `model_experiment.yaml`

A **model experiment**: shared `defaults` plus an explicit list of **runs**, each with a `run_id` overriding only the keys it changes — so "same params, different bundle" and "same bundle, different hyperparameters" cost a single line, with no duplicated config. Each run still fans out over the seed sweep. The `subset` selects which role to use (`development` for CV, `holdout`, or `all`). **HPO is a separate config** ([`hpo.yaml`](hpo.md)), not here. See [Stage 4 · Model Training](../design/06-model-training.md) and [Reports](../design/11-reports.md).

```yaml title="model_experiment.yaml"
--8<-- "model_experiment.yaml"
```
