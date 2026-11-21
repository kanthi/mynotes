# NixCon 2025: operating NixOS in production

**Goal:** Collect the deployment, update, compliance, cluster and testing practices presented at NixCon 2025, and link them to the matching chapters of this book.

> [!NOTE]
> Based on the published talk abstracts. Snippets are our own illustrations of the described techniques.

## Deployment tools

**Introducing NixOps4** (Robert Hensing) presents the successor to NixOps, asks why NixOps failed, and whether NixOps4 will replace Terraform/OpenTofu. Slides: `nix run github:roberth/nixcon-2025-slides`. Compare with [deploy-rs](../07-ops-and-fleet/03-deploy-rs.md), [Colmena](../07-ops-and-fleet/04-colmena.md) and the [Terraform boundary](../07-ops-and-fleet/07-terraform-boundary.md).

## OTA updates with systemd

Jacek Galowicz's lightning talk describes an A/B update mechanism using only features **already upstream** in NixOS and systemd:

| Piece | Role |
|-------|------|
| NixOS | Reproducible, small, hardened, immutable images |
| `systemd-repart` | Defines the A/B partition layout from the NixOS configuration |
| `systemd-sysupdate` | On device: fetch, deploy and atomically activate a new version, with rollback on boot failure |

Target audience: embedded, IoT and appliance systems. See also [Lab P3: cross-compiled edge OTA and sysupdate](../14-production-labs/12-lab-p3-cross-compiled-edge-ota-and-sysupdate.md).

## Cyber Resilience Act and long-term support

Two talks address the EU CRA, which does not bind open source itself but binds **commercial products that ship NixOS**:

| Topic | Points |
|-------|--------|
| Is NixOS ready for the CRA? | Reproducibility positions Nix well; open gaps are the **update mechanism, SBOM tooling matched to CVEs, and support durations** |
| CRA without re-certification (Ctrl-OS) | Offer long-term support for specific releases and a minimal, test-gated backport stream, so OS upgrades do not trigger re-certification. Ctrl-OS: five-year support, CVE tracking with continuous patch delivery, SLAs |

Operator takeaways: pick one release per product line, track CVEs per profile ([Vulnerability inventory](../05-services-and-security/12-vuln-inventory-across-profiles.md)), generate an SBOM at build time, and document a support end date.

## Kubernetes, and life without it

| Talk | Summary |
|------|---------|
| Kubernetes on Nix | Production cluster from the NixOS Kubernetes modules: which certificates are needed, deploying them with agenix, and configuring ingress and cert-manager through the **addon-manager** so the whole cluster lives in Nix |
| Life without Kubernetes: mirrors with NixOS | A university mirror site moved to NixOS on two machines and one maintainer; less code and maintenance than Kubernetes for that scale |

Decision rule: with a handful of hosts and one maintainer, NixOS modules plus a deploy tool beat a cluster. See [Kubernetes manifests as modules](../11-containers-and-kubernetes/02-kubernetes-manifests-as-modules.md).

## Testing and CI

| Talk | Summary |
|------|---------|
| Mastering NixOS integration tests (workshop) | Multi-host network simulation, **lightweight VMs**, interactive mode for debugging, new driver debugging features, avoiding flaky tests |
| Continuously Integrating Your Nix Config (workshop) | Pipeline that builds the config, runs tests and linters on every change |
| Local-first GitHub Actions with Nix | CI that differs little from the local environment |
| nixos-compose (garnix) | CLI to start one or more VMs from a flake, SSH in, connect them and reach them from the host: a docker-compose analogue for NixOS |

```bash
# minimal CI shape: the same commands locally and in CI
nix flake check
nix build .#nixosConfigurations.lab.config.system.build.toplevel
```

See [NixOS tests](../07-ops-and-fleet/08-nixos-tests.md), [More tests](../07-ops-and-fleet/09-more-tests.md) and [CI pipeline](../07-ops-and-fleet/10-ci-pipeline.md).

## Embedded products

"Nix as a solution to embedded Linux" (Óli) describes building the Katla synthesizer: dev environment, CI/CD, kernel optimisations and cross-platform compilation for a team spread across the world. The lesson is the same as the OTA talk: one flake builds the image, the dev shell and the CI check.
