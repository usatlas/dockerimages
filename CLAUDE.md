# Instructions for AI Coding Agents

This is a monorepo for USATLAS/MaNIAC Lab container images. Each image lives under `images/<name>/`.

## Repo layout

```
images/
  analysisbase-dask/    # ATLAS AnalysisBase + Dask + Scikit-HEP (pip + uv)
  analysisbase-dask-uc/ # UChicago AF layer over analysis-dask-base (plain Dockerfile)
  ml-platform/          # ML platform: TF/Keras + ROOT + CUDA (pixi + conda-forge)
.github/
  workflows/
    _build-and-push.yml        # Reusable workflow (don't call directly)
    analysisbase-dask.yml
    analysisbase-dask-uc.yml
    ml-platform.yml
```

## Critical rules

### CI / workflows

- The shared reusable workflow is `_build-and-push.yml`. Per-image workflows call it with `uses: ./.github/workflows/_build-and-push.yml`.
- Releases are triggered by `workflow_dispatch` with a `version` input — **no release git tags** in this repo. The SHA tag is always attached for traceability.
- `analysis-dask-uc` rebuilds automatically via `workflow_run` when `analysis-dask-base` builds on `main`. Do not reinstate `repository_dispatch` for this.
- Path filters on each workflow are intentional — modifying `_build-and-push.yml` triggers all three builds.
- Never hardcode secrets. Use `secrets: inherit` and the documented secret names (`DOCKER_HUB_USERNAME`, `DOCKER_HUB_PASSWORD`, `OSG_HARBOR_USERNAME`, `OSG_HARBOR_PASSWORD`).

### Per-image tooling

`analysisbase-dask` uses **pip + uv** with a hashed lockfile (`docker/requirements.lock`). To update dependencies:
```bash
cd images/analysisbase-dask && make lock
```

`ml-platform` uses **pixi**. Always commit `pixi.toml` and `pixi.lock` together:
```bash
cd images/ml-platform
# edit pixi.toml
CONDA_OVERRIDE_CUDA=12.6 pixi install  # regenerate lock
git add pixi.toml pixi.lock
```

Dockerfile commands in `ml-platform`'s final stage **must** be prefixed with `/app/entrypoint.sh` to activate the pixi environment.

### Version management for ml-platform

```bash
cd images/ml-platform
pixi run -e dev bump        # updates pixi.toml + tbump.toml with today's date
# commit pixi.toml + tbump.toml, then:
gh workflow run ml-platform.yml -f version=YYYY.M.D
```

### Local test builds

```bash
# analysisbase-dask
cd images/analysisbase-dask && make build

# ml-platform
docker build --platform linux/amd64 --build-arg ENVIRONMENT=ml -t ml-platform:test images/ml-platform/
docker run --rm ml-platform:test python -c "import tensorflow, keras; print('OK')"
```

## When to ask before proceeding

- Changing any base image version
- Adding a new registry or modifying registry configuration
- Changing CUDA version in ml-platform
- Adding a new image to the monorepo (requires workflow + README updates)
- Making breaking changes to the shared `_build-and-push.yml` reusable workflow
