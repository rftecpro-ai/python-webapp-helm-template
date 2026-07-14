# python-webapp-helm-template

A minimal Flask "Hello World" web app, containerized with Docker and deployed
to Kubernetes via a Helm chart and ArgoCD GitOps. This is the Helm-based
sibling of [python-webapp-template](https://github.com/rftecpro-ai/python-webapp-template),
which uses Kustomize instead.

## Repository structure

```
.
├── app.py                        # Flask app (single "/" route, returns JSON)
├── conftest.py                   # empty; makes pytest resolve `app` as an importable module
├── tests/
│   └── test_app.py               # pytest suite for app.py
├── requirements.txt               # runtime dependencies (flask, gunicorn)
├── requirements-dev.txt           # requirements.txt + pytest, for local dev/CI
├── Dockerfile                     # multi-arch image build, served via gunicorn
├── VERSION                        # human-edited release version (see CI/CD below)
├── .github/workflows/ci.yml       # test → build-and-push → update-manifest pipeline
├── chart/                         # Helm chart ArgoCD deploys
│   ├── Chart.yaml                 # chart metadata (name, version)
│   ├── values.yaml                 # shared defaults (image repo, replicas, ports)
│   ├── values-dev.yaml             # dev-environment override (deployed image tag)
│   └── templates/
│       ├── deployment.yaml         # Deployment template (2 replicas, port 8000)
│       └── service.yaml            # ClusterIP Service template (80 → 8000)
├── applications/
│   └── argocd-app.yaml            # ArgoCD Application CRD, registers chart/ with ArgoCD
├── .dockerignore                  # excludes tests/, chart/, docs, etc. from the image build context
├── .gitignore                     # excludes __pycache__/, .venv/, .pytest_cache/, etc.
└── README.md
```

Not shown above: `__pycache__/` and `.pytest_cache/` are ephemeral, gitignored
directories that Python/pytest regenerate locally — they're not part of the
repo. `.git/` is version-control internals.

See [CI/CD](#cicd) and [Deployment (ArgoCD / Kubernetes)](#deployment-argocd--kubernetes)
below for how these pieces fit together.

## Local development

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements-dev.txt

python app.py          # runs on http://localhost:8000
pytest                 # run tests
```

## Docker

```bash
docker build -t ffribeiro/python-webapp-helm-template:local .
docker run -p 8000:8000 ffribeiro/python-webapp-helm-template:local
```

## CI/CD

`.github/workflows/ci.yml` runs on every push/PR to `main`:

- **test** — installs dependencies and runs `pytest`.
- **build-and-push** — on push to `main` only (after tests pass), builds a
  multi-arch (`linux/amd64`, `linux/arm64`) image and pushes it to Docker Hub
  as `ffribeiro/python-webapp-helm-template:<VERSION>` and `:<commit-sha>`.
- **update-manifest** — after the image is pushed, sets `image.tag` in
  [`chart/values-dev.yaml`](chart/values-dev.yaml) to `<commit-sha>` and
  commits that change back to `main` (with a skip-ci marker, so it doesn't
  re-trigger the pipeline). ArgoCD watches this same repo and syncs the `dev`
  environment automatically — every push to `main` results in that exact
  commit's image running in the cluster. There's no separate GitOps repo and
  no `values-prod.yaml` yet; see [Deploying to prod](#deploying-to-prod)
  below for how a `prod` environment would be added.

### Releasing a new version

Every merge to `main` already ships a `:<commit-sha>` image to `dev` via the
pipeline above. Bumping `VERSION` is only needed when you want that build to
also carry a stable, human-meaningful Docker Hub tag (e.g. `1.1`) instead of
just the commit SHA.

1. Branch off `main` for the change, e.g. `release/1.1` (matching the version
   you're about to ship makes the intent obvious in the branch list, though
   the pipeline doesn't require this naming).
2. Bump [`VERSION`](VERSION) to match (e.g. `1.0` → `1.1`). Without this, the
   `:<VERSION>` tag stays the same and just gets re-pushed pointing at the
   new commit — the `:<commit-sha>` tag still changes correctly.
3. Make the code change and commit both together.
4. Open a PR from `release/1.1` against `main` so `test` gates it before merge.
5. Merge to `main`. CI runs `test` → `build-and-push` → `update-manifest`,
   pushing the new image and bumping `chart/values-dev.yaml`'s tag.
6. ArgoCD picks up the change and syncs the `dev` `Application` automatically
   — no manual `kubectl apply` needed.
7. Verify the rollout in `dev` (`kubectl get pods`, confirm the image tag, hit
   the app, or check ArgoCD's sync status) before trusting the release.

`VERSION` and the code change are the only files you edit for a release.
[`chart/values.yaml`](chart/values.yaml)'s `image.tag` is just a chart
default and is never deployed as-is (`values-dev.yaml` always overrides it).
[`chart/values-dev.yaml`](chart/values-dev.yaml)'s `image.tag` is overwritten
automatically by `update-manifest` on every merge to `main`, using the commit
SHA — not the `VERSION` value. Any manual edit there gets clobbered on the
next merge anyway.

There's no `prod` environment in this repo today (see
[Deploying to prod](#deploying-to-prod) below) — if one is added as a
`values-prod.yaml` + ArgoCD `Application`, promoting to it should stay a
separate, deliberate step (e.g. a PR bumping `values-prod.yaml`'s tag to a
verified `dev` version), not something the pipeline does automatically.

### Required repository secrets

These jobs need secrets set in the GitHub repo
(**Settings → Secrets and variables → Actions**):

| Secret | Used by | Value |
|---|---|---|
| `DOCKERHUB_USERNAME` | `build-and-push` | Your Docker Hub username |
| `DOCKERHUB_TOKEN` | `build-and-push` | A Docker Hub [access token](https://hub.docker.com/settings/security) (not your password) |

`update-manifest` doesn't need a secret here — it pushes back to this same
repo using the default `GITHUB_TOKEN` (see
[Required repository permissions](#required-repository-permissions) below).

The right permissions needs to create before merging to `main`, or the `build-and-push` job will fail at the login step.

### Required repository permissions

`update-manifest` pushes a commit back to this repo using the default
`GITHUB_TOKEN`. If that push fails with a permissions error, enable
**Settings → Actions → General → Workflow permissions → Read and write
permissions** for this repo.

## Deployment (ArgoCD / Kubernetes)

Kubernetes manifests are packaged as a Helm chart, with per-environment
values files — adding another environment later (e.g. `prod`) is just a new
`values-<env>.yaml`, no template duplication:

```
.
├── chart/
│   ├── Chart.yaml            # chart metadata
│   ├── values.yaml            # shared defaults
│   ├── values-dev.yaml        # dev-environment overrides (image tag)
│   └── templates/
│       ├── deployment.yaml    # Deployment (2 replicas, container port 8000)
│       └── service.yaml       # ClusterIP Service, port 80 → container port 8000
└── applications/
    └── argocd-app.yaml         # ArgoCD Application CRD, points at chart/ + values-dev.yaml
```

| Path | Purpose |
|---|---|
| [`chart/templates/deployment.yaml`](chart/templates/deployment.yaml) | Deployment (2 replicas, container port 8000) |
| [`chart/templates/service.yaml`](chart/templates/service.yaml) | ClusterIP Service, port 80 → container port 8000 |
| [`chart/values.yaml`](chart/values.yaml) | Shared defaults for all environments |
| [`chart/values-dev.yaml`](chart/values-dev.yaml) | Sets the deployed image tag for `dev` |
| [`applications/argocd-app.yaml`](applications/argocd-app.yaml) | ArgoCD `Application` CRD, watches `chart/` with `values-dev.yaml` layered on top |

ArgoCD auto-detects the Helm chart from `Chart.yaml` at the watched path — the
`Application`'s `spec.source.helm.valueFiles` tells it which values file to
layer on top of the chart's defaults. Sync policy is automated with
`selfHeal: true` and `prune: true` — this repo is the source of truth. Manual
changes made directly against the cluster will be reverted, and resources
removed from the chart will be deleted from the cluster.

**One-time bootstrap** (run once against your cluster to register the app —
not something I can run for you, since it targets your local lab):

```bash
kubectl apply -f applications/argocd-app.yaml
```

After that, ArgoCD manages everything: CI's `update-manifest` job updates
`chart/values-dev.yaml`'s image tag on every push to `main`, ArgoCD detects
the drift, and syncs it to the cluster automatically.

**Infrastructure changes** (replicas, resource limits, ports): edit
[`chart/values.yaml`](chart/values.yaml) or the templates directly — these
are shared across all environments — and merge to `main`.
**Per-environment changes**: edit the relevant `values-<env>.yaml` instead.
Don't `kubectl apply` by hand either way, ArgoCD will just revert it.

## Deploying to prod

There's no `prod` environment in this repo yet, only `chart/values-dev.yaml`.
Setting one up is a one-time bootstrap; after that, promoting a build to prod
is a repeatable, deliberate step, kept separate from the `main`-triggered CI
pipeline above.

### One-time setup

1. Add `chart/values-prod.yaml`, modeled on
   [`chart/values-dev.yaml`](chart/values-dev.yaml): same structure, but its
   own `image.tag` pinned to a verified starting version (plus any other
   prod-specific overrides, e.g. replica count).
2. Add a second ArgoCD Application (e.g.
   `applications/argocd-app-prod.yaml`), a copy of
   [`applications/argocd-app.yaml`](applications/argocd-app.yaml) with its
   own `metadata.name` and `spec.source.helm.valueFiles` pointed at
   `values-prod.yaml` instead of `values-dev.yaml`.
3. Register it once: `kubectl apply -f applications/argocd-app-prod.yaml`.
4. Decide sync policy deliberately. Unlike `dev`, you likely don't want
   prod's ArgoCD Application to auto-sync from every commit — consider
   manual sync (omit `automated` from the sync policy), so a `dev`-only
   change can't silently roll out to prod.

### Promoting a build

1. Confirm the image tag currently running in `dev` is the one you want in
   prod (check `chart/values-dev.yaml`'s `image.tag`, `kubectl get pods -n
   dev`, or ArgoCD's UI).
2. Open a PR that bumps `chart/values-prod.yaml`'s `image.tag` to that same
   tag. This is the only file that changes — `chart/values.yaml` and the
   templates stay shared, and nothing in CI does this for you.
3. Get the PR reviewed and merged to `main`.
4. Sync prod: if the prod Application uses manual sync, trigger it from the
   ArgoCD UI/CLI (`argocd app sync <prod-app-name>`); if automated, ArgoCD
   picks it up on its own.
5. Verify the rollout in prod the same way as dev: `kubectl get pods -n
   prod`, confirm the image tag, hit the app, or check ArgoCD's sync status.

The `:<VERSION>` and `:<commit-sha>` images built by CI are *candidates* —
promoting one to prod is a manual, reviewed decision about *which* candidate
is production-ready, not an automatic consequence of merging to `main`.
