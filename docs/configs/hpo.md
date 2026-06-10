# HPO — `hpo.yaml`

Hyperparameter search over **one bundle**, kept separate from the [model experiment](model_experiment.md). Hundreds of trials, zero extra config files. Outputs are **segregated** under `results/experiments/{name}/hpo/` with their own index. The intended flow: search → promote the best `top_n` → seed-sweep them in a model experiment. See [Stage 4 · Model Training](../design/06-model-training.md#hyperparameter-optimization).

**Key fields**

- `method` — `grid` or `bayesian` (Optuna / TPE).
- `n_trials` — number of search trials.
- `space` — the search space (ranges, `log` scale where useful).
- `keep_checkpoints` — `all` / `top_n` / `none`; HPO models are rarely revisited.
- `top_n` — how many to promote to a seed sweep.

```yaml title="hpo.yaml"
--8<-- "hpo.yaml"
```
