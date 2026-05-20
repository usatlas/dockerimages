# dockerimages-usatlas

Monorepo for USATLAS/MaNIAC Lab container images. Each image lives in `images/<name>/` with its own Dockerfile and build tooling. Shared CI scaffolding is in `.github/workflows/`.

## Images

| Directory | Image | Purpose | Registries |
|---|---|---|---|
| [`images/analysisbase-dask/`](images/analysisbase-dask/) | `sslhep/analysis-dask-base` | ATLAS AnalysisBase 25.2.X + Dask + Scikit-HEP + JupyterLab | DockerHub, OSG Harbor |
| [`images/analysisbase-dask-uc/`](images/analysisbase-dask-uc/) | `sslhep/analysis-dask-uc` | UChicago AF wrapper over `analysis-dask-base` | DockerHub, OSG Harbor |
| [`images/ml-platform/`](images/ml-platform/) | `maniaclab/ml-platform` | ML platform: TF/Keras + ROOT + Jupyter + CUDA | DockerHub, ghcr.io, OSG Harbor |

`analysis-dask-uc` is rebuilt automatically whenever `analysis-dask-base` is built on `main` (via `workflow_run`).

## Releasing an image

Releases are triggered by `workflow_dispatch` — there are no release-trigger git tags in this repo. The image SHA tag is always attached for traceability.

```bash
# Release analysis-dask-base
gh workflow run analysisbase-dask.yml -f version=25.2.27

# analysis-dask-uc rebuilds automatically after the base.
# To trigger manually against a specific base tag:
gh workflow run analysisbase-dask-uc.yml -f base_tag=sha-abc1234

# Release ml-platform (update version in pixi.toml first)
cd images/ml-platform && pixi run -e dev bump  # updates pixi.toml + tbump.toml
cd ../..
git add images/ml-platform/pixi.toml images/ml-platform/tbump.toml
git commit -m "chore(ml-platform): bump version to 2026.5.20"
git push
gh workflow run ml-platform.yml -f version=2026.5.20
```

Tags produced per trigger:

| Trigger | Tags applied |
|---|---|
| Push to `main` | `latest`, `sha-<short>` |
| `workflow_dispatch` with `version` | `latest`, `sha-<short>`, `<version>` |
| Pull request | build only (no push) |

## Required secrets

Set these in the repository's **Settings → Secrets and variables → Actions**:

| Secret | Used for |
|---|---|
| `DOCKER_HUB_USERNAME` | DockerHub login |
| `DOCKER_HUB_PASSWORD` | DockerHub login |
| `OSG_HARBOR_USERNAME` | OSG Harbor login |
| `OSG_HARBOR_PASSWORD` | OSG Harbor login |
| `GITHUB_TOKEN` | ghcr.io login (auto-provided by GitHub) |

## CI structure

```
.github/
├── workflows/
│   ├── _build-and-push.yml          # Reusable: buildx, logins, metadata, push
│   ├── analysisbase-dask.yml        # Paths-filtered; workflow_dispatch release
│   ├── analysisbase-dask-uc.yml     # Paths-filtered + workflow_run from base
│   └── ml-platform.yml              # Paths-filtered; workflow_dispatch release
```

Each per-image workflow uses path filters so only the changed image (plus shared workflow changes) triggers a build. Modifying `_build-and-push.yml` triggers all three.

## Adding a new image

1. Create `images/<name>/Dockerfile` (and supporting files).
2. Add `images/<name>/README.md` describing the image.
3. Create `.github/workflows/<name>.yml` following the pattern of an existing workflow.
4. Add the image to the table above.
5. If it depends on another image in this repo, add a `workflow_run` trigger referencing the parent's workflow name.

## Local development

Each image subdirectory has its own `README.md` and (where applicable) `Makefile` for local builds. See:

- [`images/analysisbase-dask/README.md`](images/analysisbase-dask/README.md)
- [`images/analysisbase-dask-uc/README.md`](images/analysisbase-dask-uc/README.md)
- [`images/ml-platform/README.md`](images/ml-platform/README.md) and [`CONTRIBUTING.md`](images/ml-platform/CONTRIBUTING.md)
