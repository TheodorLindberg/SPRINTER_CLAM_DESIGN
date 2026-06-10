# Base config — `base.yaml`

Shared **roots** and rarely-edited settings, layered under every stage config so paths and registry locations live in **one place** — changing where bundles or results are written is a single edit, not a copy across every stage config.

Stage configs are merged on top (base first, stage values win): `--configfile base.yaml preprocessing.yaml`. They reference these roots instead of hard-coding paths.

**Key fields**

- `roots` — directory roots for each pipeline stage's inputs/outputs.
- `registries` — locations of the [cohort](cohorts.md) and [seed](seeds.md) registries.
- `defaults` — values inherited unless a stage overrides them.

```yaml title="base.yaml"
--8<-- "base.yaml"
```
