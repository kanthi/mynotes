---
title: 'deploy-rs 1.0 in 2026: Tagged Pins, Groups, and Multi-Profile App Deploys'
---

# deploy-rs 1.0 in 2026: Tagged Pins, Groups, and Multi-Profile App Deploys

For six years after its 2020 launch, `deploy-rs` had no version tags and users had to track `master`. On October 19, 2026, Serokell tagged **`deploy-rs v1.0.0`** (and published it to `crates.io`). The boring default for `deploy-rs` on NixOS 26.05 is: **pin `github:serokell/deploy-rs/v1.0.0` in your flake inputs, run the CLI from that flake input so `deploy` and target `activate` binaries never drift, use `--groups` and `--test` for staged rollouts, and separate fast-moving application profiles from the server's `system` profile when app teams deploy independently**.

## Mental model

`deploy-rs` is a thin Rust orchestrator over `nix` commands and two binaries: `deploy` (on the operator machine or CI runner) and `activate` (copied to the target inside the profile closure to switch symlinks and run the magic-rollback watchdog).

```text
flake.nix (inputs.deploy-rs.url = "github:serokell/deploy-rs/v1.0.0")
    │
    ├── Profile 1: system   (activate.nixos)   ──► /nix/var/nix/profiles/system
    ├── Profile 2: app-bin  (activate.custom)  ──► /nix/var/nix/profiles/per-user/app/desk-api
    └── Profile 3: user-svc (systemd user unit)──► ~/.config/systemd/user/desk-web.service
```

### What changed in `deploy-rs 1.0.0` and the 2020–2026 Nix stack

| Feature / Change | What It Does in 2026 | Operational Rule |
|---|---|---|
| **Tagged release (`v1.0.0`)** | First semantic git tag + `crates.io` release (`cargo install deploy-rs --locked`, Rust 1.85+) | Pin `github:serokell/deploy-rs/v1.0.0` in `flake.nix`; still run `deploy` from the flake so `deploy` matches target `activate` |
| **Parallel `remoteBuild` (#271)** | Builds and pushes to multiple target hosts in parallel when `remoteBuild = true` | Local builds remain sequential to protect laptop RAM; profile activation remains sequential |
| **Deployment `--groups` (#365)** | Group nodes or profiles and deploy a slice with `deploy --groups canary` | Use groups for staged rollouts or to push only app profiles without touching `system` |
| **`--test` mode (#353)** | Activates configuration without adding a boot entry (like `nixos-rebuild test`) | Use `deploy --test` for risky kernel/network experiments that a reboot should discard |
| **Magic rollback race fix (#385) & fast abort (#373)** | Fixes the fast-confirm race that rolled back healthy deploys; aborts immediately on activation failure instead of waiting out `confirmTimeout` | Safe to keep `magicRollback = true` across low-latency LAN and fast local VMs |
| **CI & target log UX (#391, #401, #330)** | Auto-disables progress bars in CI / `nom` (`--no-progress`), prefixes remote output with `📠`, and runs `deployChecks` once per flake instead of once per node | Cleaner GitHub Actions / GitLab CI logs and faster multi-node check stages |
| **CA derivations (#375) & `interactiveSudo` (#257)** | Supports content-addressed store paths and interactive sudo password prompts (`interactiveSudo = true`) with magic rollback intact | Drop root SSH login where policy requires non-root operator accounts with `sudo` |
| **Nix 2.32/2.33/2.35 & NixOS 24.11/25.11 alignment** | Parses Nix 2.32+ derivation JSON (#346, #359), benefits from Nix 2.35 lazy flake tree evaluation, and works with Rust `switch-to-configuration` and `nixos-rebuild-ng` | Keep `deploy-rs` on `v1.0.0+`; older unpinned commits break on Nix 2.32+ derivation JSON |

## Worked examples

### Case 1: Pinning `v1.0.0`, `interactiveSudo`, and `--test` / `--no-progress`

Pin the `v1.0.0` tag explicitly and configure non-root deployment via `interactiveSudo`:

```nix
# flake.nix
{
  description = "Desk deploy-rs 1.0.0 configuration";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-26.05";
    deploy-rs = {
      url = "github:serokell/deploy-rs/v1.0.0";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = { self, nixpkgs, deploy-rs }:
    let
      system = "x86_64-linux";
      pkgs = nixpkgs.legacyPackages.${system};
      deployPkgs = import nixpkgs {
        inherit system;
        overlays = [
          deploy-rs.overlays.default
          (self: super: {
            deploy-rs = {
              inherit (pkgs) deploy-rs;
              lib = super.deploy-rs.lib;
            };
          })
        ];
      };
    in
    {
      nixosConfigurations.web-01 = nixpkgs.lib.nixosSystem {
        inherit system;
        modules = [ ./hosts/web-01.nix ];
      };

      deploy.nodes.web-01 = {
        hostname = "10.100.10.41";
        sshUser = "ops";
        interactiveSudo = true;
        remoteBuild = true;
        profiles.system = {
          user = "root";
          path = deployPkgs.deploy-rs.lib.activate.nixos self.nixosConfigurations.web-01;
        };
      };

      checks = builtins.mapAttrs
        (system: deployLib: deployLib.deployChecks self.deploy)
        deploy-rs.lib;
    };
}
```

Test activation without making it the boot default (`--test`), and strip progress bars in CI (`--no-progress`):

```bash
# Ephemeral test activation (like nixos-rebuild test)
nix run .#deploy-rs -- .#web-01 --test

# CI-friendly invocation (target lines are marked with 📠)
nix run .#deploy-rs -- .#web-01 --no-progress
```

### Case 2: Serokell Pattern A — Infrastructure repo owns the systemd module, App CI deploys the binary profile

In Serokell's production infrastructure (such as `tzbot` in `gemini-infra`), decoupling app releases from full OS rebuilds follows one of two patterns.

In **Pattern A**, the infrastructure repository defines the NixOS `systemd` service and points `ExecStart` at a secondary Nix profile path (`/nix/var/nix/profiles/per-user/deskapp/current/bin/desk-api`). The app repository's CI deploys **only** the binary into that profile and restarts the service:

```nix
# 1. In the infra repo (hosts/web-01.nix): systemd unit reads from the app profile symlink
{
  users.users.deskapp = {
    isSystemUser = true;
    group = "deskapp";
    createHome = true;
    home = "/var/lib/deskapp";
  };
  users.groups.deskapp = { };

  # Allow deskapp to restart its own unit after profile switch
  security.sudo.extraRules = [
    {
      users = [ "deskapp" ];
      commands = [
        { command = "/run/current-system/sw/bin/systemctl restart desk-api.service"; options = [ "NOPASSWD" ]; }
      ];
    }
  ];

  systemd.services.desk-api = {
    wantedBy = [ "multi-user.target" ];
    after = [ "network.target" ];
    serviceConfig = {
      User = "deskapp";
      Group = "deskapp";
      ExecStart = "/nix/var/nix/profiles/per-user/deskapp/app/bin/desk-api";
      Restart = "always";
    };
  };
}
```

```nix
# 2. In the app repo (flake.nix): deploy ONLY the binary profile + restart hook
deploy.nodes.web-01 = {
  hostname = "10.100.10.41";
  profiles.app = {
    sshUser = "deskapp";
    user = "deskapp";
    profilePath = "/nix/var/nix/profiles/per-user/deskapp/app";
    path = deployPkgs.deploy-rs.lib.activate.custom
      self.packages.x86_64-linux.desk-api
      "sudo /run/current-system/sw/bin/systemctl restart desk-api.service";
  };
};
```

*Trade-off:* App CI never needs root SSH access and never rebuilds the OS closure. However, changing a systemd setting (such as adding an environment variable or changing `MemoryMax`) requires a commit in the infrastructure repo and a full server redeploy.

### Case 3: Serokell Pattern B — App repo builds and deploys both the binary and a user-level systemd unit

When application developers change environment variables or flags frequently and you want **zero** touches to the server's `system` profile, the app repository builds the `systemd` unit file alongside the binary and deploys both as a user-level systemd service profile (`linger` enabled for the service user on the host):

```nix
# In the app repo (flake.nix): bundle binary + generated user service unit
let
  appPkg = self.packages.x86_64-linux.desk-api;

  unitFile = pkgs.writeTextDir "systemd/user/desk-api.service" ''
    [Unit]
    Description=Desk API User-Level Service

    [Service]
    Environment="LOG_LEVEL=info"
    Environment="LISTEN_ADDR=127.0.0.1:8080"
    ExecStart=${appPkg}/bin/desk-api
    Restart=always

    [Install]
    WantedBy=default.target
  '';

  serviceBundle = pkgs.symlinkJoin {
    name = "desk-api-service-bundle";
    paths = [ appPkg unitFile ];
  };

  activateUserService = pkgs.writeShellScript "activate-desk-api" ''
    set -euo pipefail
    mkdir -p ~/.config/systemd/user
    ln -sfn "$PROFILE/systemd/user/desk-api.service" ~/.config/systemd/user/desk-api.service
    XDG_RUNTIME_DIR="/run/user/$(id -u)" systemctl --user daemon-reload
    XDG_RUNTIME_DIR="/run/user/$(id -u)" systemctl --user enable --now desk-api.service
    XDG_RUNTIME_DIR="/run/user/$(id -u)" systemctl --user restart desk-api.service
  '';
in
{
  deploy.nodes.web-01 = {
    hostname = "10.100.10.41";
    profiles.desk-api = {
      sshUser = "deskapp";
      user = "deskapp";
      path = deployPkgs.deploy-rs.lib.activate.custom
        serviceBundle
        (builtins.toString activateUserService);
    };
  };
}
```

On the NixOS server, the infrastructure repo only needs `users.users.deskapp.linger = true;` and a reverse proxy forwarding to `127.0.0.1:8080`. App settings changes now deploy straight from the app repo without touching the infrastructure flake.

### Case 4: Multi-architecture cross-deploy workaround (`--remote-build --skip-checks`)

One remaining sharp edge documented in the `1.0.0` release notes (#216): when deploying an `x86_64-linux` NixOS server from an `aarch64-darwin` MacBook (or vice versa) using `--remote-build`, `deploy-rs` still runs `nix flake check` **locally** before connecting to the remote builder, which fails on architecture mismatch.

Use `--remote-build --skip-checks` from the cross-architecture laptop while letting Linux CI enforce `nix flake check`:

```bash
deploy .#web-01 --remote-build --skip-checks
```

And if your SSH key requires a hardware YubiKey PIN or passphrase (#225 — `deploy-rs` opens multiple SSH connections per run), enable SSH connection multiplexing (`ControlMaster auto`, `ControlPersist 60s`) for the target host so you tap the token only once per deployment.

## The trap

The trap is **installing `deploy-rs` globally via `cargo install deploy-rs` and letting it drift from the `inputs.deploy-rs` version in your flake**. Because the target node runs the `activate` binary built from your flake input while your laptop runs `deploy`, version skew between the two binaries can break CLI flag handshakes between `deploy` and `activate`. Always invoke `deploy` from the flake (`nix run .#deploy-rs` or `pkgs.deploy-rs` inside `devShells.default`).

The second trap is **upgrading Nix on your workstation to Nix 2.32+ while keeping an old pre-2026 commit of `deploy-rs` in `flake.lock`**. Nix 2.32 and 2.33 changed the JSON output format of `nix show-derivation` / `nix derivation show`, which breaks older `deploy-rs` commits immediately. Pin `v1.0.0` or newer.

## The boring rule

- Pin `github:serokell/deploy-rs/v1.0.0` in `flake.nix` and run `deploy` from your flake's `devShell`, not an unpinned `crates.io` binary.
- Use `deploy --test` before committing to boot-default changes on risky remote edits.
- Choose **Pattern A** (infra owns unit, app deploys binary profile) when ops controls service sandboxing; choose **Pattern B** (app deploys user systemd unit + binary) when app teams iterate rapidly on service flags and environment variables.
- Enable SSH `ControlMaster` / `ControlPersist` when deploying with hardware security keys so `deploy-rs`'s multiple SSH sessions only prompt once.

## Try this

1. Update a flake input to `github:serokell/deploy-rs/v1.0.0` and run `deploy --help` to inspect the `--groups`, `--test`, and `--no-progress` flags.
2. Deploy a test change with `deploy .#lab-vm --test`, verify `/run/current-system` switched, reboot the VM, and confirm it returns to the previous boot-default generation.
3. Build a custom profile using `deploy-rs.lib.activate.custom` for `pkgs.hello` into `/nix/var/nix/profiles/per-user/$USER/hello-test` without touching the `system` profile.
4. Configure `~/.ssh/config` with `ControlMaster auto` and `ControlPersist 60s` for a lab VM and verify a full `deploy` opens only a single underlying TCP/SSH authentication session.
