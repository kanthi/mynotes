# deploy-rs: field notes for my fleet

**Goal:** One page I can re-read before pinning, upgrading or using `deploy-rs`: what to pin, what changed, how to split system and app deploys, what still hurts, and when plain `nixos-rebuild` is enough.

> [!NOTE]
> Rewritten in my own words from a November 2026 maintainer retrospective on the tool. Deeper treatment with full examples lives in [deploy-rs 1.0 in 2026](../07-ops-and-fleet/03b-deploy-rs-2026-and-app-profiles.md). This chapter is the short decision sheet.

## What the tool is

`deploy-rs` deploys **Nix profiles** from a flake to remote machines over SSH. A profile can be:

| Profile kind | Example |
|--------------|---------|
| NixOS system | The whole host configuration |
| Application | One binary or service closure |
| User profile | A per-user service or dotfile set |

If a deploy fails, or the target cannot confirm it, **magic rollback** restores the previous profile. That safety net, plus several profiles and hosts in one run, is why I use it instead of a bare rebuild. See [deploy-rs](../07-ops-and-fleet/03-deploy-rs.md).

## Pin it, and run the CLI from the flake

For years there were no versions and the only option was to follow the main branch. Since the first tagged release, 1.0.0 (19 October 2026), I can pin a tag and read a changelog.

```nix
{
  inputs.deploy-rs.url = "github:serokell/deploy-rs/v1.0.0";
}
```

| Rule | Reason |
|------|--------|
| Use the `deploy` CLI from the **flake input** | The `activate` program on the target is built from that input, and a CLI of another version may not match it |
| `cargo install deploy-rs --locked` is for using it as a Rust library | Needs Rust 1.85 or newer, and Nix is still required to deploy |
| Old crate versions (0.1.2 and below) were a different project | The name was handed over, so 1.0.0 is the first release of this tool |

## What is new since the early years

Recent additions I should know about:

| Feature | Use |
|---------|-----|
| Parallel build and push with `remoteBuild` | Faster multi-host deploys. Local builds stay sequential and activation is still one at a time |
| Groups and `--groups` | Deploy only part of a flake: staged rollouts |
| `--test` | Like `nixos-rebuild test`: activate without making it the boot default |
| Rollback race fixed | A fast confirmation could be missed and a good deploy rolled back |
| Fail-fast activation | Stops waiting at once instead of sitting out the timeout |
| Better errors | A failed Nix command shows the command and its output |
| `--no-progress` | Progress bars switch off by themselves in CI logs and when piped into a log monitor; target output is visibly marked |
| Checks once per flake | Not once per target |
| Content-addressed derivations | Supported in profiles |

Older features that were never announced and are easy to miss:

| Since | Feature |
|-------|---------|
| 2021 | Several nodes in one command; nodes that succeeded roll back if a later one fails. Use as a Rust library |
| 2022 | `remoteBuild`: build on the target with its own substituters |
| 2023 | `activate.darwin` for nix-darwin; fix for user profiles after Nix moved them |
| 2024 | `interactiveSudo` for a non-root SSH user with magic rollback intact; CI deploys every change to VMs |

## Why the tool needs upkeep

It is a thin layer over Nix commands, so every change in their output reaches it.

- **Flakes are still experimental** (stabilisation is planned, with no date), so Nix commands can change between versions.
- **Nix 2.32 and 2.33 changed the derivation JSON** that deploy-rs parses. Deploys broke on the new Nix until fixes landed. My rule: after upgrading Nix on the workstation, run a deploy to a test host before touching the fleet. The latest Nix changes are in [Nix 2.36 and the official installer](13-nix-2-36-release-and-installer.md).
- **There is more than one Nix.** A fork of Nix 2.18 exists (first release July 2024) and one vendor ships its own distribution. The fork is not officially supported by deploy-rs, though some people run it.
- **Nix 2.35 no longer always copies a flake's source into the store before evaluation**, so large repositories may evaluate faster. Not measured by the authors; I should measure mine.

## When `nixos-rebuild` is enough

Two neighbouring programs were rewritten: `switch-to-configuration` now has a Rust version (default since NixOS 24.11, works with deploy-rs unchanged), and `nixos-rebuild` is now the Python `nixos-rebuild-ng` (default since NixOS 25.11).

```bash
# one host, one system configuration
nixos-rebuild switch --flake .#web01 --target-host root@web01
```

| Need | Use |
|------|-----|
| One machine, one NixOS config | `nixos-rebuild --target-host` |
| Several hosts or profiles, user profiles, magic rollback | deploy-rs |
| Pull-based, machines behind NAT | A pull tool such as comin, see [NixCon 2024 highlights](12-nixcon-2024-highlights.md) |

## Two ways to deploy an app

deploy-rs can deploy any Nix profile, not only a NixOS system, which leads to two layouts.

| | **System-owned** | **App-owned** |
|--|------------------|---------------|
| Where the service unit lives | App repository exports a NixOS module; the infrastructure repository imports it as a flake input | App repository builds the unit |
| What CI deploys | Only the app binary, as its own profile; activation restarts the service | The unit's closure, as a **user-level** service, with an activation script that links the unit and restarts it |
| Changing an environment variable | Update the flake lock in the infrastructure repo and redeploy the **whole server** | Redeploy the app only |
| Cost | Little setup per app | More setup in each app repository |
| Best for | Stable services owned by the platform team | Fast-moving apps owned by app teams |

A sketch of the system-owned layout:

```nix
{
  deploy.nodes.web01 = {
    hostname = "web01.example.invalid";
    sshUser = "deploy";
    user = "root";
    interactiveSudo = true;
    profiles = {
      system.path = deploy-rs.lib.x86_64-linux.activate.nixos
        self.nixosConfigurations.web01;
      app = {
        user = "app";
        path = deploy-rs.lib.x86_64-linux.activate.custom
          self.packages.x86_64-linux.app
          "systemctl restart app.service";
      };
    };
  };
}
```

Option and helper names change, so check the current manual before copying. See also [Shared modules](../07-ops-and-fleet/05-shared-modules.md) and [CI pipeline](../07-ops-and-fleet/10-ci-pipeline.md).

## Known limits and workarounds

| Limit | Workaround today |
|-------|------------------|
| Activation is sequential (parallel deploy is the most requested feature) | Group hosts and accept the order |
| Deploying NixOS from a Mac, or the reverse, is hard: flake checks run locally even with remote build | `--remote-build --skip-checks` |
| Several SSH connections per deploy, so a PIN or passphrase is asked repeatedly | Use an agent or SSH connection multiplexing |
| No diff between old and new configuration before deploying | Compare yourself: [Diffing configurations](../07-ops-and-fleet/15c-nix-diff-between-revisions.md) and [Read an update before switching](../07-ops-and-fleet/15-read-an-update-before-switching.md) |
| Large backlog of open issues and pull requests | Prefer small, focused pull requests; expect no dates |

## Companion habit: keep inputs fresh

The maintainers also use a tool that updates flake inputs across many repositories and opens pull requests. The idea is worth copying: a pinned `flake.lock` is only safe if something proposes updates regularly. See [Is my flake lock stale?](../07-ops-and-fleet/16-is-my-flake-lock-stale.md).

## Checklist before a fleet deploy

- [ ] `deploy-rs` input pinned to a tag; CLI from the flake
- [ ] Nix version on the workstation tested against a throwaway host
- [ ] Rollout staged with groups, `--test` first for risky changes
- [ ] Magic rollback left on; SSH access verified from the deploy machine
- [ ] Diff reviewed
- [ ] Decision recorded: system-owned or app-owned per service
