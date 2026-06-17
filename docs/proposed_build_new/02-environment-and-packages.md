# 02 · Environment & packages

Implements [`docs/environment.md`](../environment.md): a self-contained Apptainer SIF for
production, SIF + mounted `uv` venv for development, and the HF/SSL/CUDA env vars for HPC — now
as a **uv workspace** so each stage package resolves together but installs and versions on its
own.

## Python & the workspace

- **Python 3.11** *(build choice — matches the torch/timm/VALIS support window; 3.12 once VALIS
  ships wheels)*.
- **uv workspace.** The root `pyproject.toml` declares the members; one `uv.lock` resolves every
  component together, so versions never diverge across packages, yet each component is its own
  distribution with its own dependency list. In-repo dependencies are **path/workspace
  sources**, so editing `shared` is instantly visible to every stage without a republish.

```toml
# ROOT pyproject.toml
[tool.uv.workspace]
members = ["components/*"]

[tool.uv.sources]                     # in-repo deps resolve to the workspace, not PyPI
histomil-shared = { workspace = true }
```

```toml
# components/shared/pyproject.toml  — the contract layer; intentionally light
[project]
name = "histomil-shared"
requires-python = ">=3.11,<3.12"
dependencies = [
  "pydantic>=2.6", "pandera>=0.18",
  "pandas>=2.2", "pyarrow>=15", "numpy>=1.26",
  "h5py>=3.10", "pyyaml>=6", "shapely>=2.0",
  "structlog>=24", "gitpython>=3.1", "typer>=0.12",
]
# heavy/optional capabilities are extras so each stage pulls only what it needs:
[project.optional-dependencies]
wsi     = ["openslide-python>=1.3", "tifffile>=2024.2", "zarr>=2.17", "opencv-python>=4.9", "scikit-image>=0.22"]
torch   = ["torch>=2.2", "timm>=0.9", "huggingface_hub>=0.21"]   # model defs in shared.models
reports = ["plotly>=5.20", "jinja2>=3.1"]                        # toolkit in shared.report
```

```toml
# components/preprocessing/pyproject.toml  — representative stage package
[project]
name = "histomil-preprocessing"
requires-python = ">=3.11,<3.12"
dependencies = [
  "histomil-shared[wsi,torch]",        # the only in-repo dep
  "torch>=2.2", "timm>=0.9", "huggingface_hub>=0.21",
]
[project.scripts]
histomil-preprocess = "histomil.preprocessing.cli:app"
```

Each stage's full dependency set:

| Component | Beyond `histomil-shared` | Console script |
|---|---|---|
| `wsi-transformation` | `valis-wsi` (+ native libvips), `opencv-python`, `scikit-image` | `histomil-wsi` |
| `preprocessing` | `torch`, `timm`, `huggingface_hub`, `openslide-python`, `tifffile`, `zarr` (+ `conch`) | `histomil-preprocess` |
| `training` | `torch`, `scikit-learn`, `optuna` | `histomil-train` |
| `evaluation` | `torch` | `histomil-evaluate` |
| `heatmaps` | `matplotlib`, `opencv-python`, `shapely` | `histomil-heatmap` |
| `reporting` | `histomil-shared[reports]` (plotly, jinja2) | `histomil-report` |
| `shared` | (extras: `wsi`, `torch`, `reports`) | — |

> `preprocessing` also pulls `histomil-shared[reports]` because `resolve_cohort` emits the cohort
> HTML report via the shared toolkit — see [07 · star-constraint consequence](07-reports.md#where-the-toolkit-lives-a-star-constraint-consequence).

> **Native deps** (libvips for VALIS, openslide, libtiff) are installed at the OS layer in the
> SIF (`apt`), not via pip — `pyproject` declares only the Python bindings. That is why
> `wsi-transformation` and the embedding side are SIF concerns, not a pure `uv add`.

## Why this packaging matches the goal

- **Independent updates.** Bumping the training loop touches only `histomil-training` and its
  lock entries; preprocessing and evaluation are untouched and don't rebuild.
- **Independent install / environments.** A reporting box installs `histomil-reporting`
  (plotly + jinja2, no torch). A login-node prefetch box installs only `histomil-preprocessing`.
  CI's contract tests install `histomil-shared` alone.
- **One resolution.** Because it's a single workspace lock, the torch pinned for training is the
  same torch pinned for evaluation — no drift, despite separate packages.

## Concern → package map

| Concern | Package(s) | Lives in |
|---|---|---|
| Config models + layering | `pydantic`, `pyyaml` | `shared.config` |
| Table schemas / validation | `pandera`, `pandas`, `pyarrow` | `shared.schemas` |
| Binary IO + formats (HDF5, GeoJSON, BEAM) | `h5py`, `numpy` | `shared.io`, `shared.formats` |
| Geometry / outlines | `shapely`, `opencv-python`, `scikit-image` | `shared` (extra), `wsi-transformation`, `heatmaps` |
| WSI reading | `openslide-python`, `tifffile`+`zarr` | `shared.io.wsi` |
| Registration | `valis-wsi` (+ libvips) | `wsi-transformation` |
| Embedding + MIL model defs | `torch`, `timm`, `huggingface_hub` | `shared.models`, used by `preprocessing`/`training`/`evaluation` |
| Folds / HPO | `scikit-learn`, `optuna` | `training` |
| Report toolkit | `plotly`, `jinja2` | `shared.report` (extra), used by `reporting` + `preprocessing` |
| Orchestration | `snakemake`, `snakemake-executor-plugin-slurm` | `workflow/` (dev/CI dep) |

## Containers

```text
containers/histomil.def        # Apptainer recipe (self-contained production SIF)
containers/build.sh            # apptainer build histomil.sif histomil.def
```

Recipe outline:

1. **Base** — CUDA + cuDNN runtime matching the cluster driver.
2. **System libs** — `libvips`, `openslide-tools`, `libtiff`, `libgl1` via `apt`.
3. **uv + workspace** — copy the root `pyproject.toml`, every `components/*/pyproject.toml`, and
   `uv.lock`; `uv sync --frozen --all-packages` to bake all components + their extras into one
   venv.
4. **Entrypoint** — activate the venv; every `histomil-*` console script and `snakemake` on
   `PATH`.

A SIF carries **all** components, since one cluster run spans stages. The packaging split buys
independent *development and release*, not separate runtime images.

**Dev vs production** (the decided approach): production runs the self-contained SIF; dev
bind-mounts a `uv` workspace venv over the baked one (`uv sync --all-packages` locally) so
`uv add` into any component iterates without a rebuild, then `build.sh` promotes.

## Environment variables (HPC)

| Variable | Purpose | Where |
|---|---|---|
| `HF_TOKEN` | gated model download (UNI2-h, CONCH, GigaPath) | `scripts/prefetch_weights.py` on a **login** node, once |
| `HF_HUB_OFFLINE=1` | no network on compute nodes | every `embed` / `infer` worker |
| `HF_HUB_DISABLE_SSL_VERIFICATION=1`, unset `SSL_CERT_FILE` | incomplete container CA bundle | first download |
| `CUDA_VISIBLE_DEVICES` | GPU selection (SLURM-set) | `embed`, `train_run`, `infer` |

`scripts/prefetch_weights.py` (ships with `histomil-preprocessing`) populates the shared HF cache
before any offline GPU job — the failure mode in
[`impl/preprocessing.md`](../impl/preprocessing.md).

## Tooling

- **`ruff`** — lint + format across the workspace.
- **`mypy`** — strict on `histomil-shared` (the contract layer) and each component's `schemas`
  usage; looser where torch/valis stubs are thin.
- **`pytest`** — per-component `tests/` + top-level `tests/integration/`; see
  [08 · Testing](08-testing-and-roadmap.md).
