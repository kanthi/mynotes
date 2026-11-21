# NixCon 2024 highlights

**Goal:** Keep the lasting ideas from NixCon 2024 (Berlin, 25-27 October 2024) that are still useful when operating NixOS today.

> [!NOTE]
> Source: the public [NixCon 2024 schedule](https://talks.nixcon.org/nixcon-2024/schedule/), 43 sessions, using abstracts only. For later years see [NixCon 2025](05-nixcon-2025-field-guide.md) and [NixCon 2026](10-nixcon-2026-core-builds-and-stores.md).

## Deployment and fleet management

| Talk | Idea |
|------|------|
| **Comin: GitOps for NixOS machines** (lewo) | A **pull-mode** deployer: each machine polls Git, deploys its own configuration, supports testing branches, multiple remotes (no single point of failure), machine migrations and Prometheus metrics |
| **Scalable and secure NixOS deploys on AWS** (Arian van Putten) | A company deploying dozens of times a day: your own S3 cache, custom AMIs, Auto Scaling Groups with Instance Refresh for zero-downtime rolling releases, AWS Systems Manager instead of NixOps for long-lived stateful hosts |
| **Call-by-hash** (Julian Kirsten Arni) | Give NixOS machines hash-derived names so they talk to each other only by hash: addresses zero-downtime and "transactional" infrastructure deploys |
| **Thymis** | A web front end for managing several NixOS devices |
| **Clan** (Jörg Thalheim, Kenji Berthold) | Self-hosting made easy: VPN integration, backups, secrets |
| **NixOS Facter** (Brian McGee) | Declarative hardware detection for NixOS |
| **Deploy NixOS remotely with nixos-anywhere and disko** (workshop) | Install NixOS with ZFS on a remote Ubuntu host |

Compare: [deploy-rs](../07-ops-and-fleet/03-deploy-rs.md), [Colmena](../07-ops-and-fleet/04-colmena.md), [disko](../07-ops-and-fleet/01-disko.md), [automatic upgrades](../07-ops-and-fleet/14-auto-upgrade.md). Pull-mode tools like comin fit fleets behind NAT, where push tools cannot reach.

## Modules and architecture

**Enabling incremental adoption of NixOS with module contracts** (Pierre Penninckx): most NixOS modules assume they own their runtime dependencies (the Nextcloud module installs its own database), so you cannot point it at a database on another host without forking. Contracts let a module declare *what* it needs and let any provider satisfy it. Use this thinking when you write shared modules: see [Shared modules](../07-ops-and-fleet/05-shared-modules.md).

## Build systems and CI

- **buildbot-nix** (Richard Brežák): Nix-native CI based on Buildbot: zero configuration for flake repositories, parallel evaluation, binary cache integration, failure caching, GitHub and Gitea integration, deployments via Hercules-CI effects. See [CI pipeline](../07-ops-and-fleet/10-ci-pipeline.md).
- **Nix and Bazel** and **Integrating Nix and Buck2** (Aleksander Gondek et al., Claudio Bley): the premise is full reproducibility plus fast incremental builds; the practice is surprises and design clashes. Buck2 integration brings Nix packages in as fine-grained dependencies via flakes.
- **nix-make** (Tobias Mayer): a Nix-native build system.
- **FUSEd with Nix**: attach a shared `/nix/store` to container images through FUSE, giving images as small as about 500 KB. A forerunner of the lazy-store talks in [NixCon 2026](10-nixcon-2026-core-builds-and-stores.md).
- **Beyond Docker: native Nix images** (Edgar Lee) and **Kubenix** (Pim Kunis): container images and Kubernetes manifests generated from Nix modules. See [Kubernetes manifests as modules](../11-containers-and-kubernetes/02-kubernetes-manifests-as-modules.md).

## Supply chain security

| Talk | Idea |
|------|------|
| **rebuilding builders instead of trusting trust** (Martin Schwaighofer) | Add builder provenance to signatures: from a "built it myself" flag up to source links and remote attestation of the builder |
| **Signed Nix Expressions** (Finn Landweber) | Machine-in-the-middle attacks on Nixpkgs fetches; verify Git signatures, learning from Guix and RFC 100 |
| **hashes all the way down** (Martin Schwaighofer) | Clear terminology for how build inputs are hashed, separate from how store contents are addressed |
| **Pwning the entire Nix ecosystem** | A security-research lightning talk on ecosystem attack surface |

## Core tooling

- **Tvix** (flokli) and **devenv is switching to Tvix** (Domen Kožar): a Rust reimplementation of Nix and a developer-environment tool moving off C++ Nix.
- **The C API is Surprisingly Fun** (Thomas Bereknyei, Yannik Sander): build your own utilities on the stable Nix C API.
- **Developer environment with devenv.sh** (workshop): convert a PostgreSQL plus Rust project from docker-compose into devenv.

## Operations and small-scale use

- **Monitoring your system using NixOS** (João Thallis): Prometheus and Grafana configured declaratively. See [Observability patterns](../05-services-and-security/10-observability.md).
- **Deterministic firmware with Nix** (Óli): reproducible firmware builds with hardware-in-the-loop testing in CI.
- **Nix as the Swiss Army Knife in Cloud Development Environments**, **Nix as a static site generator** and **nix-bitcoin** (deploying Bitcoin nodes with krops): entry-level and unusual uses that show how far a derivation stretches.
- **Deploying random AI models with dream2nix** (Yorick van Pelt) and **Nixpkgs CUDA news**: Python and GPU software packaged reproducibly.

## Community

State of the Union 2024 (cache, finances, infrastructure, survey), the **Nix Constitutional Assembly** governance panel, **Becoming a Nixpkgs Contributor**, and the **Summer of Nix** mentorship experience report.
