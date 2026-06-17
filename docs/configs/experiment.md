# Experiment config — `experiments/<name>.yaml`

One file per experiment: shared `defaults` plus an explicit list of **runs**, each with a `run_id` overriding only the keys it changes and fanning out over the seed sweep — so "same params, different bundle" and "same bundle, different hyperparameters" each cost one line. An optional `hpo` block defines a separate search whose best trials are promoted into the sweep. *What* to evaluate or visualize is a command-line target, not a config field. See [Stage 4 · Model Training](../design/06-model-training.md).

**Key fields**

- `defaults` — inherited by every run: `subset` (`development` (CV) / `holdout` / `all`), `target`, `seed_set`, `architecture` (`family` → `type` → `params`), `hyperparameters`, `label_balancing` (per fold, train split only), `target_normalization` (per-fold train-split stats, de-normalized at evaluation; default on).
- `runs` — explicit variations; each needs a `run_id` and references its `bundle` by **components** (cohort · stain · variant · embedding), from which the bundle id is derived (no hand-typed compound ids).
- `hpo` — optional separate search (`grid` | Optuna/TPE); promote the best `promote_top_n` into the seed sweep. Outputs are segregated and top-N kept (see [reports](pipeline.md)).

!!! note "Cohort consistency"
    A run's `bundle` implies its cohort; the `seed_set` must reference the same cohort (validated).

```yaml title="experiment.yaml"
--8<-- "experiment.yaml"
```
