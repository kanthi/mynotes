---
title: 'Project: Multi-Cluster GitOps IDP with Argo CD ApplicationSets & Crossplane'
---

# Project: Multi-Cluster GitOps IDP with Argo CD ApplicationSets & Crossplane

Build a self-service **Internal Developer Platform (IDP)** where a single Git PR provisions cloud or local infrastructure via **Crossplane XRDs/Compositions** (or OpenTofu), bootstraps target clusters with **Argo CD ApplicationSets** using deterministic sync waves, injects short-lived OIDC secrets (zero static keys), enforces **Kyverno** admission policies, and wires Prometheus/Grafana alerts with linked Markdown runbooks.

## Platform Flow

```text
Developer PR (app-claim.yaml)
        |
        v  GitHub Actions (Kyverno/Trivy validation)
Argo CD ApplicationSet (Git Generator -> multi-cluster fan-out)
   ├── Sync Wave -1: Crossplane XRD/Composition (RDS/S3/Namespace + Python Composition Function)
   ├── Sync Wave  0: ExternalSecrets (OIDC Workload Identity) + RBAC + NetworkPolicy
   ├── Sync Wave  1: Application Deployment + ServiceMonitor
   └── Sync Wave  2: PrometheusRule alerts with runbook_url -> docs/runbooks/<alert>.md
```

## Argo CD ApplicationSet with Sync Waves

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: golden-path-apps
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/org/platform-gitops.git
        revision: HEAD
        directories:
          - path: apps/*/*
  template:
    metadata:
      name: '{{path[1]}}-{{path[2]}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/org/platform-gitops.git
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        name: '{{path[1]}}'
        namespace: '{{path[2]}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

## Acceptance Drills

- Open a PR adding `apps/staging/orders/claim.yaml`; verify Crossplane provisions the database/bucket before the app pod starts (sync wave ordering).
- Introduce manual drift (`kubectl scale` or edited ConfigMap) and verify Argo CD self-heals and fires a drift notification.
- Attempt to deploy an unsigned or `runAsRoot` image and verify Kyverno blocks admission.
