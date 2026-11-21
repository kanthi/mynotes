# NixCon 2025: configuration patterns, secrets and sandboxing

**Goal:** Capture the configuration-layout, secrets, sandboxing and alternative-system ideas from NixCon 2025.

> [!NOTE]
> Based on the published talk abstracts. The snippet below is our own sketch of the technique described.

## Layout: the dendritic pattern

Shahar "Dawn" Or's workshop teaches the [dendritic pattern](https://github.com/mightyiam/dendritic) for reorganising NixOS, nix-darwin and Home Manager configurations, with tools to make refactoring safer. Compare with [Flake layout](../04-home-and-flake-layout/00-index.md) and [Shared modules](../07-ops-and-fleet/05-shared-modules.md).

## Secrets and external configuration

| Talk | Idea |
|------|------|
| When Not to Nix (Ryota) | Keep JSON/YAML/TOML in separate files; mind "file content vs file as path"; use sops-nix to substitute secret values into that file |
| agenix-shell | Inject secret environment variables into flake `devShells` using age, easing new developer onboarding without an external secrets manager |

A sketch of the first technique with sops-nix templates:

```nix
{ config, ... }:
{
  sops.secrets."app/api_key" = { };

  sops.templates."app.toml".content = ''
    [server]
    listen = "127.0.0.1:8080"
    api_key = "${config.sops.placeholder."app/api_key"}"
  '';
  # the service reads config.sops.templates."app.toml".path at runtime
}
```

The secret value never enters the Nix store; only the placeholder does. See [sops-nix](../05-services-and-security/02-sops-nix.md).

## Supply chain and sandboxing

- **Supply Chain Security Panel** (John Ericson, Julien Malka, Arian van Putten, Martin Schwaighofer; moderated by Dawn Or): Nix promises much, but pieces are missing in implementation, adoption or UX. A good prompt for your own threat model: [The secrets problem](../05-services-and-security/01-secrets-problem.md) and [Host hardening](../05-services-and-security/04-hardening.md).
- **jail.nix** (Alex David): a library that wraps derivations in **bubblewrap** using high-level combinators (give network access, allow Wayland rendering) instead of hand-written flags. Motivation: memory-unsafe software and supply-chain attacks. See [Agent sandboxes](../13-platform-engineering/12-agent-sandboxes-loop-engineering-and-overlays.md).

## Packaging

**Python packaging with nixpkgs, pyproject.nix and uv2nix** (adisbladis) walks from plain nixpkgs to pyproject.nix and uv2nix, after a primer on `requirements.txt`, `setup.py`, `pyproject.toml` and Conda. Rule of thumb from the abstract's framing: pick by how much of the lock file you want Nix to understand. See [Part 6: Packaging](../06-packaging/).

## Desktop and multi-machine tools

**Flatpaks the Nix way** presents [nix-flatpak](https://github.com/gmodena/nix-flatpak), which manages Flatpak apps declaratively. The trade-off discussed is **convergent state versus full reproducibility**.

```nix
{
  imports = [ inputs.nix-flatpak.nixosModules.nix-flatpak ];
  services.flatpak.enable = true;
  services.flatpak.packages = [ "org.mozilla.firefox" ];
}
```

**The Road Towards a NixOS UI** (clan.lol): a 3D visual interface that generates Nix modules; network-wide modules with roles (client, server), secret generators, support for several VPNs at once with the best chosen at deploy time, and nix-darwin integration. **What is Clan?** was a lightning intro to the same project.

## Alternative systems

**finix** (aanderse) is an experimental Nix-built OS that uses **finit as PID 1** and reuses much NixOS scaffolding. It favours bold experiments (alternative service frameworks, minimal module sets) over stability and is the speaker's daily driver. The argument: sibling projects, including a Nix-based BSD, can test ideas that may flow back to NixOS. Compare with the [systemd-based model](../08-internals/00-index.md) the rest of this book teaches.

## Hardware

NovaCustom (a sponsor) sells coreboot (Dasharo) laptops and mini PCs, preinstalled with NixOS on request, and invited feedback on NixOS compatibility. Recreational Receipt Printing packaged receipt-printer drivers for Nix as a reminder that any driver can be packaged.
