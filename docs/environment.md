# Environment & Tooling

How the pipeline is packaged and run: the container, environment variables, launch scripts, and a quick TissUUmaps viewer.

## Container

The pipeline runs inside an **Apptainer/Singularity `.sif`** image; Snakemake rules enter it via `container:` directives, so the controller and workers share one environment.

- **Production — self-contained SIF.** Build an image with **all Python packages installed inside it**. The result is immutable and reproducible, needs no network at runtime, and pins exact versions — the right choice for cluster runs and published results.
- **Development — SIF + a mounted `uv` venv.** Rather than rebuilding the image for every dependency change, mount a [`uv`](https://docs.astral.sh/uv/)-managed virtualenv into the SIF. This is the **decided dev approach**: `uv` makes it fast to **add and test packages** (`uv add …` / `uv pip install …`) without a rebuild, so iteration stays quick. When a change is ready to promote, bake it into the SIF for production.

!!! tip "Promotion path"
    Develop against the mounted uv venv → once the dependency set is settled, rebuild the SIF with those packages baked in → run production from the self-contained image.

## Environment variables

| Variable | Purpose |
|---|---|
| `HF_TOKEN` | Access **gated** HuggingFace models (UNI2-h, CONCH, GigaPath). Set before first download. |
| `HF_HUB_OFFLINE=1` | Run fully offline after models are cached (cluster nodes without network). |
| `HF_HUB_DISABLE_SSL_VERIFICATION=1`, `unset SSL_CERT_FILE` | Workaround when the container CA bundle is incomplete and HF/PyTorch downloads fail on SSL. |
| `CUDA_VISIBLE_DEVICES` | GPU selection (set by SLURM); the embedding and training steps respect it. |

## Running (SLURM)

A launch script submits the Snakemake **controller** to SLURM; the controller then dispatches **one worker job per rule** (cluster-generic executor / profile), each running inside the container. Heavy rules (`embed`, `train_run`) request a GPU.

```bash
# example — submit a named target
sbatch scripts/run.sh <target> [configfile=config/<stage>.yaml]
# e.g.  sbatch scripts/run.sh embeddings configfile=config/pipeline.yaml
```

Targets are the [workflow named targets](impl/workflow.md#named-targets) (`register`, `cohort`, `embeddings`, `train`, …).

## TissUUmaps with Docker

For viewing outlines / patch grids / attention GeoJSON layers locally:

```bash
docker run --rm -p 5100:5100 -v /path/to/data:/mnt tissuumaps/tissuumaps
# then open http://localhost:5100 and browse /mnt
```
