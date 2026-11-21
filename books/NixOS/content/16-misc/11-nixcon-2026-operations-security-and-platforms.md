# NixCon 2026: operations, security and platforms

**Goal:** Collect the NixCon 2026 talks on running NixOS: compliance, hardening, networking, deployment state, testing, corporate adoption and new hardware.

> [!NOTE]
> Based on the published abstracts of the [NixCon 2026 schedule](https://talks.nixcon.org/nixcon-2026/schedule/). Tools named here (for example `nixos-secrets`, `declarative-runtime`, `netbox-nixos`) are early projects. Read their repositories before depending on them.

## Compliance and long-term support

| Talk | Key points |
|------|------------|
| **Nix and the EU Cyber Resilience Act** (Florian Pester) | From **December 2027**, commercial products with digital elements need SBOMs, vulnerability disclosure, patch lifecycle commitments and incident reporting. Nix's technical foundations are strong; the organisational ones are not. The talk compares how FreeBSD and Zephyr organised (for example a Foundation acting as Open Source Software Steward) |
| **When the Honeymoon Ends** (Martin Messer) | Supporting a NixOS product for **five years**: pinning to a release is not enough after community support ends, and CVE tracking is required |
| **Implementing the NixOS STIG** (Alexander Foremny) | A DISA-style Security Technical Implementation Guide for NixOS exists since 2025: **103 controls** over auditing, cryptography, SSH and kernel hardening. It ships declarative configuration with each requirement, so it can be packaged into reusable modules instead of checklists and bash scripts |

Combine these with [Host hardening](../05-services-and-security/04-hardening.md) and the [Vulnerability inventory](../05-services-and-security/12-vuln-inventory-across-profiles.md). Earlier talks: [Production operations](07-nixcon-2025-production-operations.md).

## Networking

- **Peering Declaratively** (Dionysis Grigoropoulos): run a small autonomous system on NixOS: obtaining an ASN and IPv6 space, generating service configuration in Nix, and testing routing behaviour with **nixosTests before it reaches a live upstream**. Read with [Internet-scale routing with BIRD and BGP](../05-services-and-security/05b-internet-routing-bird-bgp.md).
- **Untangling the undead spaghetti** (Molly Miller): a datacentre platform that still runs NixOS **scripted networking** because systemd-networkd could not meet some operational requirements. Scripted networking's days are numbered, so plan a replacement.
- **netbox-nixos** (Kierán Meinhardt, Jacek Galowicz): generates NixOS configuration from **NetBox** (devices, interfaces, IPs, VLANs) so the source of truth drives configuration instead of drifting beside it.
- **NixNet** (Benedikt Spies): declare virtual network experiments (topology, execution, cleanup) as a flake for reproducible research.

## Deployment state and control planes

| Talk | Idea |
|------|------|
| **The State NixOS Won't Manage** (Alexander Foremny) | Modules stop at the systemd unit. Realms, admin users and client secrets in apps such as Keycloak and Forgejo are runtime state. The `declarative-runtime` tool aims to bootstrap that state with one command |
| **Beyond Push-vs-Pull Deployments** (nick novitski) | Proposes a Nix-based control plane for almost everything, as Kubernetes is for containers |
| **You don't need a staging environment** (Shivashis Padhi) | Integration-test multi-team microservices with isolated, reproducible environments instead of a shared stateful staging system or per-team docker-compose files |
| **Nix for Cloud Native Platform Engineering** (Arik Grahl) | Organise a platform catalogue as a Nix monorepo patterned after nixpkgs, replacing Helm templating and Dockerfiles with derivations |

See [Internal developer platform](../13-platform-engineering/01-internal-developer-platform.md) and [Helm charts from Nix](../11-containers-and-kubernetes/03-helm-charts-from-nix.md).

## Testing

- **NixOS test driver workshop** (Jacek Galowicz): faster tests with **NixOS containers**, even without KVM; CUDA inside the test sandbox; mixed Intel and ARM VMs in one test; debugging over **VSOCK**; and a new **debugger hook** for tests that fail only in CI.
- **Advanced integration test scenarios** (Lukas Beierlieb): USB passthrough tested with QEMU's emulated USB devices and nested virtualisation (usbvfiod), Xen setups, and FDO.
- **Running NixOS integration tests on real hardware** (Markus Napierkowski): the familiar test driver inside a QEMU VM controls a **real machine over the network**, for drivers, firmware and peripherals.

See [NixOS tests](../07-ops-and-fleet/08-nixos-tests.md) and [More tests](../07-ops-and-fleet/09-more-tests.md).

## Secrets

**nixos-secrets** (formerly nixos-vars; experimental, tracked as a nixpkgs pull request) aims at one unified, composable interface for secrets, the job agenix and sops-nix do today, with a CLI, a module and pluggable backends. Until it lands upstream, keep the house default from [sops-nix](../05-services-and-security/02-sops-nix.md).

## Platforms, corporate adoption and hardware

| Talk | Highlights |
|------|-----------|
| **NixOS in the Corporate Trenches** (Nicolas Goudry) | A hardened developer VM behind enterprise proxies: custom proxy packaging, EDR agent integration, air-gapped sandboxed AI |
| **Staging Ground** (lassulus, Qubasa) | clan.lol as incubator for upstream pieces: disko, nixos-anywhere, nixos-facter, vars, nix-select, wrappers, container tests |
| **NGI Forge** (Ivan Minčík et al.) | A packaging and distribution platform for the 1000+ NLnet/NGI projects, built so upstream authors need no Nix experience |
| **Serving apps commodified** (Kiara Grouwstra) | Fediversity: GUIs configuring Nix modules, portable infrastructure, beyond the GDPR's data portability right |
| **Nix and AI** (Ron Efroni) | Nix as a foundation layer for AI infrastructure, replacing fragile Python environments, CUDA path conflicts and huge Docker layers |
| **nix-singularity** (Fernando Ayats) | Nix inside Singularity and Slurm on an HPC cluster |
| **Kubernetes on RISC-V** (Bruce Gain, Liang Yuning) and **NixOS on RISC-V** (liberodark) | Cross-compilation and declarative kernels bypass slow native builds; NixOS ported to the SpacemIT K1 with a RISC-V cache |
| **Nix Badge** (Morgan Jones) | Dual-boot aarch64 and riscv64 NixOS on a Sophgo SG2000 hardware badge |
| **Embedded CUDA workshop** (Jacek Galowicz) | Jetson Orin Nano images with NixOS as a Yocto alternative: cross-compilation, custom software, hygienic evaluation |
| **Tangled** | A decentralised, AT Protocol-based code platform with Nix-powered CI |

Also on the programme: the **Official Documentation workshop** (docs.nixos.org), the **State of the Union** session, and community sessions such as speed-dating.

## Takeaways

1. Compliance (CRA, STIG) is now a normal reason to adopt NixOS: plan SBOMs and a support window early.
2. Modules cannot manage runtime state of applications: write a runbook or a bootstrap tool for it.
3. The NixOS test driver is the strongest safety net you have: use containers for speed and real-hardware mode where VMs lie.
