---
title: Sync Policies, Pruning, and Self-Healing
---

# Sync Policies, Pruning, and Self-Healing

By default, an Argo CD `Application` only **observes**: it highlights drift as `OutOfSync` and waits for an operator to click Sync or run `argocd app sync`. Enabling an automated `syncPolicy` turns Argo CD into a closed-loop GitOps reconciler—and understanding the interaction between `prune`, `selfHeal`, and `syncOptions` prevents accidental production deletions.

## Controlling Sync Behavior

| Setting | Default | What It Does | Operational Caution |
| :--- | :--- | :--- | :--- |
| `automated: {}` | Disabled | Auto-syncs whenever Git changes | Does **not** delete removed resources or revert manual `kubectl` edits |
| `automated.prune: true` | `false` | Deletes cluster objects removed from Git | Protect stateful PVCs/databases with `argocd.argoproj.io/sync-options: Prune=false` |
| `automated.selfHeal: true` | `false` | Reverts out-of-band `kubectl` edits back to Git | Blocks emergency hot-patching via `kubectl` unless auto-sync is temporarily paused |
| `ServerSideApply=true` | `false` | Uses Kubernetes Server-Side Apply instead of `last-applied-configuration` | Essential for large CRDs that exceed the 256 KiB annotation size limit |
| `PruneLast=true` | `false` | Runs resource deletions only after all new resources become healthy | Prevents downtime when renaming a ConfigMap or Service |

## Production `syncPolicy` Blueprint with Retries

```yaml
spec:
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - Validate=true
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
      - ServerSideApply=true
    retry:
      limit: 5
      backoff:
        duration: 10s
        factor: 2
        maxDuration: 3m
```

Setting `allowEmpty: false` is an important guardrail: if a misconfigured Kustomize overlay or empty Git branch suddenly renders zero manifests, Argo CD refuses to prune the entire production namespace.

> [!WARNING]
> Always annotate critical stateful resources (such as PostgreSQL `PersistentVolumeClaim` or `Cluster` objects) with `argocd.argoproj.io/sync-options: Prune=false` so an accidental manifest removal in Git cannot wipe production data.

## Hands-On Lab: Verify `Prune=false` Protection in Dry-Run

Annotate a sensitive ConfigMap or PVC manifest so Argo CD excludes it from automated pruning:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: critical-bootstrap-config
  annotations:
    argocd.argoproj.io/sync-options: Prune=false
data:
  mode: "protected"
```

After syncing the application, remove the file from Git and inspect `argocd app diff <app-name>`—the resource will appear as `OutOfSync (Prune=false)` and remain intact in the cluster even with `automated.prune: true` enabled.
