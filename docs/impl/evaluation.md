# Impl · Evaluation (BEAM generation)

Recipe for [Stage 5](../design/07-evaluation.md). [Overview](../design/07-evaluation.md) · [Specification](../spec/evaluation.md) · **Implementation**.

Reference libraries: **PyTorch** (inference), **h5py** (BEAM), **numpy/pandas**.

## Per-patient inference

```python
folds = load_fold_assignments(run, seed_set)          # patient → fold/split
for bag in bundle.bags(subset):                       # bags for the chosen subset
    pat = bag.patient_id
    if subset == "development":
        ckpts = [run.checkpoint(fold_of_test(folds, pat))]   # out-of-fold only
    elif subset == "holdout":
        ckpts = run.all_checkpoints()                        # ensemble
    else:  # all
        ckpts = [run.checkpoint("full")]

    preds, attns = [], []
    for ck in ckpts:
        model = load(run.architecture, ck)
        out = model(bag.embeddings)                   # prediction (+ attention if attention-based)
        preds.append(out.prediction)
        if model.has_attention:
            attns.append(out.attention)               # (N,) aligned to bag patches

    write_beam(bag, prediction=aggregate(preds),
               attention=mean(attns) if attns else None,
               checkpoints_used=[c.id for c in ckpts])
```

- `aggregate(preds)` — mean for `holdout`; identity for the single out-of-fold / full checkpoint.
- Attention transforms written to BEAM: `raw` (model weights), `sigmoid = σ(raw)`, `rank = percentile(raw)`.
- Omit `/attention` entirely for non-attention architectures — do not write zeros.

## Writing the BEAM

```python
with h5py.File(f"{biopsy_id}__{run_id}.beam.h5", "w") as f:
    f.attrs.update(provenance)                         # run_id, checkpoints_used, subset, …
    f["patches/coords"] = bag.coords                   # raw frame, same order as embeddings
    f["prediction"]     = prediction
    if attention is not None:
        f["attention/raw"], f["attention/sigmoid"], f["attention/rank"] = attention, sig, rnk
    write_outline(f, scan_outline)                     # polygon + quartiles
    if label is not None: write_labels(f, label)
```

HDF5 is appendable, so enrichment steps (extra heatmaps, contributions) add datasets later without rewriting.

## Notes

- Validate run stain == bundle stain before inference.
- Holdout: record every contributing checkpoint id; cross-check none trained on the patient.
- Per-quartile / multi-stain bags: aggregate per biopsy (predictions combined per architecture; attention concatenated with quartile/stain tags).
