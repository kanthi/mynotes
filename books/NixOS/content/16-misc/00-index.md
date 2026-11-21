---
title: Misc
---

# Misc

Field notes that have to stay dated. The rest of the book teaches the model. This part records what is true on a particular day, and one install that uses that pin from the first command.

**Checked:** 13 November 2026.

| Chapter | You leave with |
|---------|----------------|
| Current Nix and NixOS | A pin you can defend: Nix 2.35, NixOS 26.05, and what 26.11 is allowed to change |
| How a release is built | Who evaluates, who builds, who signs, and which URL your machine should trust |
| Minimal ISO to a desktop | A UEFI lab named `lab`, user `alice`, flake in `~/nixos-config`, Plasma on generation 2 |
| A builder that sleeps | `build-01` suspends when SSH is idle and wakes when the desk builds |
| NixCon 2025 field guide | Index of all talks, adoption and community notes |
| NixCon 2025: infrastructure and builds | Hydra, cache.nixos.org, evaluation speed, dynamic derivations |
| NixCon 2025: production operations | OTA updates, CRA, Kubernetes, testing, embedded |
| NixCon 2025: config, secrets, sandboxing | Dendritic pattern, sops templates, jail.nix, Flatpak, Clan, finix |
| NixCon 2025 deep dives | Hydra rewrite, A/B OTA, NixOps4, dendritic, jail.nix, finix, nixos-compose, LTS |
| NixCon 2026: core, builds and stores | Scaling Nix, Nixception, lazy stores, unprivileged builds, laut, `.devShell` |
| NixCon 2026: operations, security, platforms | CRA, STIG, declarative AS, test driver, nixos-secrets, RISC-V |
| NixCon 2024 highlights | comin, module contracts, buildbot-nix, supply-chain provenance, Tvix |
| Nix 2.36 and the official installer | Which installer to use, daemon and cgroup changes, what to update |
| Installing NixOS on a small VPS | Install routes that failed and worked, nixos-anywhere, newcomer pitfalls |
| Restic backups field report | Fragile rsync snapshots, Restic design, laptop timers, containers, `restic copy`, REST server |

Read the version chapter before the install. The install assumes 26.05 media and a flake input of `nixos-26.05`. When 26.11 is actually released, change the pin with the upgrade-planning runbook. Leave `system.stateVersion` where the install set it.
