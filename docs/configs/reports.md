# Reports — `reports.yaml`

Controls the report layer — a regenerable **view** over the artifacts (manifests, `runs.parquet`, BEAM). See [Reports](../design/11-reports.md).

**Key fields**

- `plots` — plotting backend; `plotly` for interactive, self-contained figures.
- `css` — the [standalone stylesheet](../design/11-reports.md#format) (no MkDocs dependency).
- `layout` — `hybrid`: self-contained per-run folders plus cross-linking index pages.
- `hpo` — [HPO segregation](../design/11-reports.md#hpo-is-kept-apart-from-the-seed-sweep): its own index and how many checkpoints to keep.
- `tables` — client-side export buttons **and** a link to the canonical backing CSV/Parquet ([data export](../design/11-reports.md#data-export)).
- `include` — which report sections to build.

```yaml title="reports.yaml"
--8<-- "reports.yaml"
```
