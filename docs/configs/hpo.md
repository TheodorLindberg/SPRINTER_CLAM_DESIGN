# HPO — `hpo.yaml`

Hyperparameter search over **one bundle**, kept separate from the [model experiment](model_experiment.md). Hundreds of trials, zero extra config files. Outputs are **segregated** under `results/experiments/{name}/hpo/` with their own index. The intended flow: search → promote the best `top_n` → seed-sweep them in a model experiment. See [Stage 4 · Model Training](../design/06-model-training.md#hyperparameter-optimization).

**Key fields**

- `method` — `grid` or `bayesian` (Optuna / TPE).
- `n_trials` — number of search trials.
- `space` — the search space (per-parameter ranges; `log: true` for log-scale sampling).
- `promote_top_n` — how many of the best trials to carry forward into a seed sweep.

!!! note "Checkpoint retention lives elsewhere"
    How many trial checkpoints are kept on disk (`all` / `top_n` / `none`) is set in [`reports.yaml`](reports.md) under `hpo.keep_checkpoints`, not here — `hpo.yaml` only decides how many trials are *promoted*.

```yaml title="hpo.yaml"
--8<-- "hpo.yaml"
```
