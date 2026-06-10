# Base config — `base.yaml`

Shared **roots** and rarely-edited settings, layered under every stage config so paths and registry locations live in **one place**. Changing an output root, or where bundles are written, is a single edit here instead of being copied across every stage config.

Stage configs are merged on top of base (base first, stage values override) — e.g. Snakemake `--configfile base.yaml preprocessing.yaml`. Stage configs reference these roots rather than hard-coding paths.

```yaml title="base.yaml"
--8<-- "base.yaml"
```
