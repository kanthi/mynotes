---
title: Application and Environment Repositories in Flux
---

# Application and Environment Repositories in Flux

Flux separates *what* you build from *where* and *how* it runs by splitting source code (`desk-api`) from cluster deployment state (`desk-infra`). That separation prevents infinite CI loops (where a manifest update retriggers an application build), keeps production RBAC distinct from developer commit access, and lets a single Git history represent the entire cluster state.

## Standard Flux Fleet Repository Layout

```text
desk-infra/
├── apps/
│   ├── base/
│   │   └── desk-api/
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       └── kustomization.yaml
│   └── overlays/
│       ├── staging/
│       │   └── kustomization.yaml
│       └── production/
│           └── kustomization.yaml
└── clusters/
    ├── staging/
    │   └── apps.yaml
    └── production/
        └── apps.yaml
```

## Wiring `GitRepository` and `Kustomization` CRDs

Flux reconciles in two distinct stages: the **source-controller** fetches and verifies a Git commit into an immutable in-cluster artifact (`GitRepository`), and the **kustomize-controller** renders and server-side-applies that artifact (`Kustomization`).

```yaml
# clusters/staging/apps.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: desk-infra
  namespace: flux-system
spec:
  interval: 1m
  url: <desk-infra-git-url>
  ref:
    branch: main
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: desk-api-staging
  namespace: flux-system
spec:
  interval: 5m
  retryInterval: 1m
  timeout: 3m
  prune: true
  wait: true
  sourceRef:
    kind: GitRepository
    name: desk-infra
  path: ./apps/overlays/staging
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: desk-api
      namespace: staging
```

> [!TIP]
> Setting `prune: true` tells Flux to delete Kubernetes objects whose YAML files are removed from Git, while `wait: true` blocks downstream dependent `Kustomization` resources until the Deployment rollout reports ready.

## Hands-On Lab: Render and Verify Kustomize Overlays Before Pushing

Validate your base and overlay manifests locally before Flux ever pulls them:

```bash
mkdir -p /tmp/flux-overlay-lab/apps/{base/desk-api,overlays/staging}

cat > /tmp/flux-overlay-lab/apps/base/desk-api/deployment.yaml <<'YAML'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: desk-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: desk-api
  template:
    metadata:
      labels:
        app: desk-api
    spec:
      containers:
        - name: api
          image: ghcr.io/example/desk-api:1.0.0
YAML

cat > /tmp/flux-overlay-lab/apps/base/desk-api/kustomization.yaml <<'YAML'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
YAML

cat > /tmp/flux-overlay-lab/apps/overlays/staging/kustomization.yaml <<'YAML'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: staging
resources:
  - ../../base/desk-api
replicas:
  - name: desk-api
    count: 2
YAML

# Render the exact YAML Flux will apply (requires kubectl or kustomize)
kubectl kustomize /tmp/flux-overlay-lab/apps/overlays/staging 2>/dev/null \
  || cat /tmp/flux-overlay-lab/apps/overlays/staging/kustomization.yaml
```
