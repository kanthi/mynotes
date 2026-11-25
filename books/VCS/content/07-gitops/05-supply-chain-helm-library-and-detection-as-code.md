---
title: 'Project: Hardened Supply-Chain Pipeline, Helm Library Chart & Detection-as-Code'
---

# Project: Hardened Supply-Chain Pipeline, Helm Library Chart & Detection-as-Code

Build an end-to-end DevSecOps delivery pipeline combining:
- A **reusable Helm library chart** enforcing non-root `securityContext`, `PodDisruptionBudget`, resource limits, and OpenTelemetry sidecar/annotations.
- **Hardened container supply-chain CI** using `ko` / BuildKit, distroless/Chainguard base images, **Cosign** keyless OIDC signing, **Syft** SBOM generation, and **Kyverno** admission policies rejecting unsigned or vulnerable images.
- **Detection-as-Code** testing for **SIGMA/YARA** rules and **osquery/FleetDM** queries in GitHub Actions.

## Supply-Chain & Detection Pipeline

```text
git push / PR
  ├── Build distroless image (CGO_ENABLED=0, non-root UID 65532)
  ├── Generate SPDX SBOM (syft) & scan CVEs (trivy --exit-code 1 --severity CRITICAL)
  ├── Sign image & attest SBOM with keyless OIDC (cosign sign / cosign attest)
  ├── Lint Terraform/OpenTofu IAM for wildcards (conftest / OPA)
  ├── Validate SIGMA/YARA & osquery detection rules against replay fixtures
  └── Deploy Helm chart -> synthetic smoke test -> auto-rollback on failure
```

## Keyless Cosign Sign + SBOM Attestation Step

```yaml
- name: Generate SBOM and sign container image
  run: |
    syft "${IMAGE}@${DIGEST}" -o spdx-json=sbom.spdx.json
    cosign sign --yes "${IMAGE}@${DIGEST}"
    cosign attest --yes --predicate sbom.spdx.json --type spdxjson "${IMAGE}@${DIGEST}"
```

## Acceptance Drills

- Push an unsigned image tag to the cluster and verify Kyverno rejects the pod with a clear policy violation message.
- Introduce a synthetic malicious fixture matching a YARA/SIGMA rule and verify the CI detection-as-code job catches it.
