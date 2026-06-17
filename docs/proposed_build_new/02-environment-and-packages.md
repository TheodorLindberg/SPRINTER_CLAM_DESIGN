# 02 · Environment & packages

Implements [`docs/environment.md`](../environment.md): a self-contained Apptainer SIF for
production, SIF + mounted `uv` venv for development, and the HF/SSL/CUDA env vars for HPC — as
**one `uv`-managed package** with optional-dependency extras per concern.

## Python & the package

- **Python 3.11** *(build choice — matches the torch/timm/VALIS support window; 3.12 once VALIS
  ships wheels)*.
- **One `pyproject.toml`, one `uv.lock`.** Heavy/optional capabilities are **extras**, so a light
  environment (report-only, schema validation, login-node prefetch) installs only what it needs,
  while the production SIF installs `histomil[all]`.

```toml
# pyproject.toml (shape)
[project]
name = "histomil"
requires-python = ">=3.11,<3.12"

dependencies = [                 # always-on core: config, manifest, schemas-as-checks, IO, ids
  "pydantic>=2.6",               # config + manifest validation (kills the YAML 1e-4 trap, extra=forbid)
  "pandas>=2.2", "pyarrow>=15", "numpy>=1.26",
  "h5py>=3.10", "pyyaml>=6", "shapely>=2.0",
  "typer>=0.12", "structlog>=24", "gitpython>=3.1",
]

[project.optional-dependencies]
wsi      = ["openslide-python>=1.3", "tifffile>=2024.2", "zarr>=2.17", "opencv-python>=4.9", "scikit-image>=0.22"]
register = ["valis-wsi>=1.0"]                                   # + native libvips (OS layer)
embed    = ["torch>=2.2", "timm>=0.9", "huggingface_hub>=0.21"] # + conch (git/manual)
train    = ["torch>=2.2", "scikit-learn>=1.4", "optuna>=3.6"]
reports  = ["plotly>=5.20", "jinja2>=3.1"]
all      = ["histomil[wsi,register,embed,train,reports]"]
dev      = ["pytest>=8", "pytest-cov", "ruff", "mypy", "import-linter>=2",
            "snakemake>=8.10", "snakemake-executor-plugin-slurm>=0.4"]

[project.scripts]
histomil = "histomil.cli:app"
```

> **Validation is `pydantic` + `checks.py`.** `pydantic` covers config + manifest (kills the YAML
> `1e-4` trap, `extra='forbid'`); dataframe contracts are a handful of explicit assertion functions
> in `histomil.shared.checks` (see [03](03-shared-core.md) / [06](06-formats-and-schemas.md)) — one
> validation idea, no schema framework.

> **Native deps** (libvips for VALIS, openslide, libtiff) install at the OS layer in the SIF
> (`apt`), not via pip — `pyproject` declares only the Python bindings. That is why registration
> and the embedding side are SIF concerns, not a pure `uv add`.

## Concern → where it lives

| Concern | Packages | Module |
|---|---|---|
| Config + manifest validation | `pydantic`, `pyyaml` | `shared.config`, `shared.manifest` |
| Dataframe invariants | `pandas` (+ explicit asserts) | `shared.checks` |
| Binary IO + formats (HDF5, GeoJSON, BEAM) | `h5py`, `numpy` | `shared.io`, `shared.formats` |
| Geometry / outlines | `shapely`, `opencv-python`, `scikit-image` | `shared` (extra), `wsi_transformation`, `heatmaps` |
| WSI reading | `openslide-python`, `tifffile`+`zarr` | `shared.io.wsi` |
| Registration | `valis-wsi` (+ libvips) | `wsi_transformation` |
| Embedding + MIL model defs | `torch`, `timm`, `huggingface_hub` | `shared.models`, used by `preprocessing`/`training`/`evaluation` |
| Folds / HPO | `scikit-learn`, `optuna` | `training` |
| Report toolkit | `plotly`, `jinja2` | `shared.report`, used by `reporting` + `preprocessing` |
| Boundary enforcement | `import-linter` | CI contract (no stage imports a sibling) |
| Orchestration | `snakemake`, `snakemake-executor-plugin-slurm` | `workflow/` |

## Why one package (not separate distributions)

A single team runs the whole DAG from one SIF; stages are never installed or released apart. So the
goal is **code separation**, not independent distribution — and that is delivered by submodule
boundaries + `import-linter`, at far lower cost than separate packages (no per-stage `pyproject`, no
workspace sources, no cross-package version skew, one CLI). If a stage ever needs an external
release cadence or has a conflicting dependency, splitting a clean single package — boundaries
already enforced — into separate distributions is mechanical.

## Containers

```text
containers/histomil.def        # Apptainer recipe (self-contained production SIF)
containers/build.sh            # apptainer build histomil.sif histomil.def
```

Recipe outline:

1. **Base** — CUDA + cuDNN runtime matching the cluster driver.
2. **System libs** — `libvips`, `openslide-tools`, `libtiff`, `libgl1` via `apt`.
3. **uv + project** — copy `pyproject.toml` + `uv.lock`; `uv sync --frozen --extra all` to bake the
   package + all extras into one venv.
4. **Entrypoint** — activate the venv; `histomil` and `snakemake` on `PATH`.

**Dev vs production** (the decided approach): production runs the self-contained SIF; dev
bind-mounts a `uv` venv over the baked one (`uv sync --extra all --extra dev` locally) so `uv add`
iterates without a rebuild, then `build.sh` promotes.

## Environment variables (HPC)

| Variable | Purpose | Where |
|---|---|---|
| `HF_TOKEN` | gated model download (UNI2-h, CONCH, GigaPath) | `scripts/prefetch_weights.py` on a **login** node, once |
| `HF_HUB_OFFLINE=1` | no network on compute nodes | every `embed` / `infer` worker |
| `HF_HUB_DISABLE_SSL_VERIFICATION=1`, unset `SSL_CERT_FILE` | incomplete container CA bundle | first download |
| `CUDA_VISIBLE_DEVICES` | GPU selection (SLURM-set) | `embed`, `train_run`, `infer` |

`scripts/prefetch_weights.py` populates the shared HF cache before any offline GPU job — the failure
mode in [`impl/preprocessing.md`](../impl/preprocessing.md).

## Tooling

- **`ruff`** — lint + format.
- **`mypy`** — strict on `histomil.shared` (the contract layer); looser where torch/valis stubs are thin.
- **`import-linter`** — enforces the stage-boundary contract in CI.
- **`pytest`** — one suite (`tests/`); see [08 · Testing](08-testing-and-roadmap.md).
