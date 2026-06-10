# Stage 4 · `model_experiment.yaml`

Shared `defaults` plus an explicit list of **runs**, each with a `run_id` overriding only the keys it changes — so "same params, different bundle" and "same bundle, different hyperparameters" each cost one line, with no duplicated config. Every run still fans out over the seed sweep. **HPO is a separate config** ([`hpo.yaml`](hpo.md)). See [Stage 4 · Model Training](../design/06-model-training.md).

**Key fields**

- `defaults` — inherited by every run; a run overrides any subset of these.
- `subset` — which role to use: `development` (CV), `holdout`, or `all` (final retrain).
- `seed_set` — the [split set](seeds.md) driving the fold × model seed sweep.
- `architecture` — `family` → `type` → `params` (e.g. CLAM attention vs. mean pooling).
- `label_balancing` — applied per fold, from the training split only.
- `use_augmented_embeddings` — whether to sample the augmented sets built in preprocessing.
- `runs` — explicit variations; each needs a `run_id` and lists only its overrides.

!!! note "Cohort consistency"
    The `bundle` implies its cohort; the `seed_set` must reference the same cohort (validated).

```yaml title="model_experiment.yaml"
--8<-- "model_experiment.yaml"
```
