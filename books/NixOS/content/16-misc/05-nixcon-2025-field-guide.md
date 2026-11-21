# NixCon 2025 field guide

**Goal:** A map of what NixCon 2025 (Rapperswil-Jona, 5-7 September 2025) covered, and where each topic lands in this book.

> [!NOTE]
> Source: the public [NixCon 2025 schedule](https://talks.nixcon.org/nixcon-2025/schedule/). These notes are built from the **published talk abstracts**, not from the recordings. Where a claim goes beyond an abstract it is marked as our own advice. Watch the recording before you rely on a talk's numbers or APIs.

Other years: [NixCon 2026](10-nixcon-2026-core-builds-and-stores.md) ([operations](11-nixcon-2026-operations-security-and-platforms.md)) and [NixCon 2024](12-nixcon-2024-highlights.md).

## How to use this chapter

The conference had 69 sessions: long and short talks, lightning talks, workshops and panels. Four chapters follow this one, grouped by theme:

| Chapter | Themes |
|---------|--------|
| [Nix infrastructure and build internals](06-nixcon-2025-infrastructure-and-builds.md) | Hydra, binary caches, sandboxing, evaluation speed, dynamic derivations, formatter |
| [Operating NixOS in production](07-nixcon-2025-production-operations.md) | Deployment, OTA updates, CRA compliance, Kubernetes, mirrors, testing, embedded |
| [Configuration patterns, secrets and sandboxing](08-nixcon-2025-config-secrets-and-sandboxing.md) | Dendritic pattern, external config, agenix-shell, jail.nix, Flatpak, Clan, finix |
| [Deep dives](09-nixcon-2025-deep-dives.md) | Hydra, OTA, NixOps4, dendritic, jail.nix, finix, nixos-compose, LTS |
| [Internet routing with BIRD](../05-services-and-security/05b-internet-routing-bird-bgp.md) | BGP, anycast |

## Talk index

| Talk | Speaker | Theme | Covered in |
|------|---------|-------|------------|
| Sustainable Nix 2025: State of the Union | Ron Efroni | Governance, infra sustainability | This chapter |
| Introducing NixOps4 | Robert Hensing | Deployment | [Operations](07-nixcon-2025-production-operations.md) |
| Flatpaks the Nix way | Gabriele Modena | Desktop | [Config](08-nixcon-2025-config-secrets-and-sandboxing.md) |
| The Nix Binary Cache and AWS | Tarus Balog et al. | cache.nixos.org | [Infrastructure](06-nixcon-2025-infrastructure-and-builds.md) |
| Supply Chain Security Panel | J. Ericson, J. Malka, A. van Putten, M. Schwaighofer | Security | [Config](08-nixcon-2025-config-secrets-and-sandboxing.md) |
| How I wish Bazel had nix develop | Srini | Dev environments | This chapter |
| jail.nix | Alex David | Sandboxing | [Config](08-nixcon-2025-config-secrets-and-sandboxing.md) |
| systemd-sysupdate + repart + NixOS | Jacek Galowicz | OTA updates | [Operations](07-nixcon-2025-production-operations.md) |
| Building a Nix Binary Cache | techknowlogick | Binary caches | [Infrastructure](06-nixcon-2025-infrastructure-and-builds.md) |
| What if GitHub Actions were local-first? | Domen Kožar | CI | [Operations](07-nixcon-2025-production-operations.md) |
| Rewriting the Hydra Queue Runner in Rust | Simon Hauser | Hydra | [Infrastructure](06-nixcon-2025-infrastructure-and-builds.md) |
| You can't spell "devshell" without "hell" | Zach Mitchell | nix develop | This chapter |
| The Road Towards a NixOS UI | Kenji Berthold | Clan | [Config](08-nixcon-2025-config-secrets-and-sandboxing.md) |
| Is NixOS ready for the CRA? | Lukas Beierlieb | Compliance | [Operations](07-nixcon-2025-production-operations.md) |
| Nix-based dev environments at Shopify | Josh Heinrichs | Adoption | This chapter |
| My first Nix Aha! | Kavisha Kumar | Learning | This chapter |
| Life without Kubernetes: Mirrors with NixOS | Sizhe Zhao | Hosting | [Operations](07-nixcon-2025-production-operations.md) |
| What can't be configured with Nix? | June Stepp | Limits | This chapter |
| The official Nix formatter | Silvan Mosberger | nixfmt | [Infrastructure](06-nixcon-2025-infrastructure-and-builds.md) |
| DerivationBuilder | John Ericson | Sandboxing internals | [Infrastructure](06-nixcon-2025-infrastructure-and-builds.md) |
| garn | Julian Kirsten Arni | Alternative frontends | [Infrastructure](06-nixcon-2025-infrastructure-and-builds.md) |
| How NixOS is built: PR to /nix/store | Dionysis Grigoropoulos | Release pipeline | [Infrastructure](06-nixcon-2025-infrastructure-and-builds.md) |
| finix | aanderse | Alternative OS | [Config](08-nixcon-2025-config-secrets-and-sandboxing.md) |
| When Not to Nix: external config and sops-nix | Ryota | Config | [Config](08-nixcon-2025-config-secrets-and-sandboxing.md) |
| Share your daemons with your peers | Yannik Sander | P2P store | [Infrastructure](06-nixcon-2025-infrastructure-and-builds.md) |
| Nix Ninja | Edgar Lee | Dynamic derivations | [Infrastructure](06-nixcon-2025-infrastructure-and-builds.md) |
| agenix-shell | Andrea Ciceri | Secrets | [Config](08-nixcon-2025-config-secrets-and-sandboxing.md) |
| CRA compliance without re-certification | Florian Pester | LTS | [Operations](07-nixcon-2025-production-operations.md) |
| Nix in the Wild | Ross Turk | Adoption | This chapter |
| Styx | David Reiss | Substitution | [Infrastructure](06-nixcon-2025-infrastructure-and-builds.md) |
| NixOS on LoongArch64 | Weijia Wang, Aleksana | Ports | [Infrastructure](06-nixcon-2025-infrastructure-and-builds.md) |
| Python packaging: pyproject.nix and uv2nix | adisbladis | Packaging | [Config](08-nixcon-2025-config-secrets-and-sandboxing.md) |
| Internet scale routing with NixOS | Yifei Sun | Networking | [BIRD chapter](../05-services-and-security/05b-internet-routing-bird-bgp.md) |
| nixos-compose | Sönke Hahn | Local VMs | [Operations](07-nixcon-2025-production-operations.md) |
| Nix as a solution to embedded Linux | Óli | Embedded | [Operations](07-nixcon-2025-production-operations.md) |
| A field guide to Nix at the Corporate | Aleksander Gondek | Adoption | This chapter |
| Embarrassingly parallel evaluations | Guillaume Maudoux | Evaluation | [Infrastructure](06-nixcon-2025-infrastructure-and-builds.md) |
| Deterministic logos with Nix | Daniel Baker | FODs | [Infrastructure](06-nixcon-2025-infrastructure-and-builds.md) |
| Kubernetes on Nix | Lux | Kubernetes | [Operations](07-nixcon-2025-production-operations.md) |
| Graph2Diff for nixpkgs build errors | Mutsuha Asada | Research | [Infrastructure](06-nixcon-2025-infrastructure-and-builds.md) |
| Mastering NixOS integration tests (workshop) | Jacek Galowicz | Testing | [Operations](07-nixcon-2025-production-operations.md) |
| Continuously Integrating Your Nix Config (workshop) | João Thallis, Claudio Neto | CI | [Operations](07-nixcon-2025-production-operations.md) |
| Dendritic pattern (workshop) | Shahar "Dawn" Or | Config layout | [Config](08-nixcon-2025-config-secrets-and-sandboxing.md) |

Sponsor lightning talks (Flox, Determinate Systems, Cachix, NovaCustom, ExpressVPN), ceremonies, the hack sessions, the improv workshop "How To Fix Infinite Recursion" and the "Consulting Driven Open Source" talk are not summarised further.

## Community, governance and adoption

### Sustainable Nix 2025

Ron Efroni's state-of-the-union session covered how the Steering Committee and the Foundation keep the ecosystem healthy: 2024 milestones from infrastructure to governance, and the lessons learned. Takeaway for operators: the binary cache, Hydra and the release process are community-run infrastructure, so pin releases and mirror what you depend on.

### Adopting Nix at work

Four talks give a consistent picture of company rollouts:

| Talk | Reported lesson |
|------|-----------------|
| Shopify | An earlier 2019 effort stalled; **devenv** renewed interest. They migrated a huge set of projects incrementally, moving to local development and a monorepo, and the majority of development now runs in Nix-based environments |
| Nix in the Wild (Ross Turk) | Interviews with teams from startups building AI workflows to large cloud orgs: why they chose Nix, how they rolled it out, common blockers |
| A field guide to Nix at the Corporate | Concrete arguments for sceptics, including the rebuttal "Docker makes Nix unnecessary" |
| My first Nix Aha! | Newcomer's path: `nix-shell`, derivations, the store, pure builds. Learning resources and mental models matter more than tool features |

Our advice, based on these: start with development shells for one team, keep the first win small, and write down the "why Nix" argument for managers and security staff separately.

### Dev environments

- **"You can't spell devshell without hell"** explains how `nix develop` works and explores faster startup, shells other than Bash, and adding packages without leaving the shell. See [Debugging and troubleshooting](../13-platform-engineering/05-debugging-troubleshooting.md) when a devshell misbehaves.
- **"How I wish Bazel had nix develop"** contrasts months of custom Bazel rules (SDK paths, direnv, LSP settings) with a few lines of `shell.nix`. If you have a Bazel monorepo, Nix can supply third-party dependencies and developer shells.

### What can't be configured with Nix?

June Stepp's short talk tests the limits of "Nix can do anything". The practical reading: state that lives outside the store (databases, device pairing, interactive logins) stays imperative. Document it as a runbook rather than forcing it into modules.
