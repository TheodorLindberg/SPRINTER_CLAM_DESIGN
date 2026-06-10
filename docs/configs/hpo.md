# HPO — `hpo.yaml`

Hyperparameter search over **one bundle**, kept separate from the [model experiment](model_experiment.md). Declares the search `space`, the optimizer (`grid` / `bayesian`), and `n_trials` — hundreds of trials, zero extra config files. Outputs are **segregated** under `results/experiments/{name}/hpo/` with their own index; `keep_checkpoints` decides how many to retain. The intended flow: HPO explores → promote the best `top_n` → run a seed sweep on them in a model experiment. See [Stage 4 · Model Training](../design/06-model-training.md) and [Reports](../design/11-reports.md).

```yaml title="hpo.yaml"
--8<-- "hpo.yaml"
```
