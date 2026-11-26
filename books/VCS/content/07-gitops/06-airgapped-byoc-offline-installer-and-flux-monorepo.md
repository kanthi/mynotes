---
title: 'Project: Air-Gapped BYOC Offline Installer & Multi-Environment Flux CD Monorepo'
---

# Project: Air-Gapped BYOC Offline Installer & Multi-Environment Flux CD Monorepo

Master two real-world enterprise delivery topologies:
1. **Air-Gapped "BYOC" Bundle & Offline Registry Installer:** Package an application's container images, Helm charts, RAG/LLM artifacts, and SBOMs into a single signed tarball using **Zarf / Helm / `skopeo`**, deploy into a network-isolated Kubernetes cluster with a local registry mirror and private PKI, and ship a **Go pre-flight diagnostic CLI** that validates VPC, DNS, RBAC, and storage before install.
2. **Multi-Environment Flux CD Monorepo:** Promote a Go + Python monorepo across `dev -> staging -> prod` using Flux `Kustomization`, `HelmRelease`, `ImageUpdateAutomation`, SOPS/ExternalSecrets, and OIDC workload identity.

## Air-Gapped Bundle Workflow

```text
Connected Build Environment:
  skopeo copy -> local OCI layout + helm package + syft SBOM + cosign sign -> byoc-bundle-v1.tar.zst

Air-Gapped Customer VPC (zero internet egress):
  1. preflight-cli check (DNS, StorageClass, RBAC, cert SANs, free disk)
  2. Load images into customer private registry (skopeo / zarf package deploy)
  3. helm upgrade --install with private registry override & custom CA bundle
  4. Run API contract + Playwright E2E smoke tests
```

## Flux CD Monorepo Layout

```text
clusters/
  dev/        # Kustomization -> apps/base + overlays/dev + ImageUpdateAutomation
  staging/    # Kustomization -> apps/base + overlays/staging
  prod/       # Kustomization -> apps/base + overlays/prod (manual PR promotion)
infrastructure/
  controllers/ (cert-manager, external-secrets, kyverno)
  configs/     (ClusterIssuer, ClusterSecretStore via OIDC)
```

## Acceptance Drills

- Disconnect the target `kind` cluster from the internet (drop egress via `iptables`/`nftables`) and verify the full BYOC bundle installs and passes E2E tests using only the local registry mirror.
- Push a new container tag and verify Flux `ImageUpdateAutomation` commits the updated tag to `dev` automatically.
