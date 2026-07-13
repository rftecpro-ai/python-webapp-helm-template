# python-webapp-helm-template

A minimal Flask "Hello World" web app, containerized with Docker and deployed
to Kubernetes via a Helm chart and ArgoCD GitOps. This is the Helm-based
sibling of [python-webapp-template](https://github.com/rftecpro-ai/python-webapp-template),
which uses Kustomize instead.

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

1. **test** — installs dependencies and runs `pytest`.
2. **build-and-push** — on push to `main` only (after tests pass), builds a
   multi-arch (`linux/amd64`, `linux/arm64`) image and pushes it to Docker Hub
   as `ffribeiro/python-webapp-helm-template:<VERSION>` and `:<commit-sha>`.
3. **update-manifest** — after the image is pushed, sets `image.tag` in
   [`chart/values-dev.yaml`](chart/values-dev.yaml) to `<commit-sha>` and
   commits that change back to `main` (with a skip-ci marker in the commit
   message, so it doesn't re-trigger the pipeline). This is what drives the
   ArgoCD deployment below — every push to `main` results in that exact
   commit's image running in the cluster.

To cut a new Docker Hub release version, bump the version in the
[`VERSION`](VERSION) file and merge to `main` — that value becomes the
`:<VERSION>` image tag. Without a bump, every merge to `main` re-pushes the
same version tag (pointing at the latest commit), so treat updating `VERSION`
as the release step. This is independent of the `:<commit-sha>` tag that
`update-manifest` deploys, which always tracks the latest commit on `main`.

`VERSION` is the **only** file you need to edit for a release.
[`chart/values.yaml`](chart/values.yaml)'s `image.tag` is just a chart
default and is never deployed as-is (`values-dev.yaml` always overrides it).
[`chart/values-dev.yaml`](chart/values-dev.yaml)'s `image.tag` is overwritten
automatically by `update-manifest` on every merge to `main`, using the commit
SHA — not the `VERSION` value. Any manual edit there gets clobbered on the
next merge anyway.

### Required repository secrets

The push step needs these secrets set in the GitHub repo
(**Settings → Secrets and variables → Actions**):

| Secret | Value |
|---|---|
| `DOCKERHUB_USERNAME` | Your Docker Hub username (`ffribeiro`) |
| `DOCKERHUB_TOKEN` | A Docker Hub [access token](https://hub.docker.com/settings/security) (not your password) |

I can't create these secrets for you (I don't have your Docker Hub credentials) —
add them manually before merging to `main`, otherwise the `build-and-push` job
will fail at the login step.

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
