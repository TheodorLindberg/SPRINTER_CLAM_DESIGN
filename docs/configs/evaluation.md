# Stage 5 · `evaluation.yaml`

Runs inference and writes one [BEAM file](../formats/beam.md) per biopsy per model. See [Stage 5 · Evaluation](../design/07-evaluation.md).

**Key fields**

- `bundle` / `model` — the prepared cohort to score and the trained run to score it with.
- `subset` — `holdout` for the final locked test, `development` for cross-validated test predictions. These are [two distinct scores](../design/02-data-model.md#cohorts-roles-and-splits).
- `attention` — which attention variants (`raw` / `sigmoid` / `rank`) to write into BEAM.
- `include_embeddings` — store embeddings in BEAM, or reference them from the bundle.

```yaml title="evaluation.yaml"
--8<-- "evaluation.yaml"
```
