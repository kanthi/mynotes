---
title: 'Lab H2: Git-SHA Edge VM, Systemd Sandboxing, and Magic Rollback'
---

# Lab H2: Git-SHA Edge VM, Systemd Sandboxing, and Magic Rollback

Field and edge appliances cannot wait for an operator to drive to a site with a keyboard when a bad firewall rule locks out SSH. The boring default for an autonomous edge node is: **stamp `system.configurationRevision` with the exact Git commit SHA (`self.rev`), lock custom daemons inside strict `systemd` namespaces, and push updates via `deploy-rs` with `magicRollback = true` so any cut to management connectivity self-heals automatically**.

## Mental model

Every NixOS generation on the edge node carries its birth certificate in `/run/current-system/nixos-version` and `nixos-version --configuration-revision`. When `deploy-rs` switches to a new generation, it activates the profile and starts a local watchdog timer on the target. Unless the operator's deployment client successfully re-establishes an SSH confirmation handshake after activation, the target's watchdog automatically reverts to the previous generation.

```text
git commit (SHA: 7f3a91c)
    │
    ├── system.configurationRevision = self.rev
    ├── services.edge-telemetry (DynamicUser, ProtectSystem=strict)
    │
    ▼
deploy --flake .#edge-01
    ├── 1. Copy closure + activate new profile
    ├── 2. Start magic-rollback watchdog timer on edge-01
    ├── 3. Re-connect via SSH to confirm health
    │        ├── [OK]      ──► disarm timer; generation 7f3a91c sticks
    │        └── [TIMEOUT] ──► watchdog reverts to previous generation!
```

## Worked examples

### Case 1: Package an isolated telemetry daemon as a flake derivation

Save a minimal Python telemetry daemon and package it inside `pkgs.writers.writePython3Bin` (or `buildPythonApplication`) so it has zero ambient dependencies outside `/nix/store`.

Save as `pkgs/edge-telemetry.nix`:

```nix
# pkgs/edge-telemetry.nix
{ pkgs }:
pkgs.writers.writePython3Bin "edge-telemetry" { } ''
  import json
  import os
  import time

  state_dir = os.environ.get("STATE_DIRECTORY", "/tmp")
  while True:
      payload = {
          "status": "healthy",
          "timestamp": int(time.time()),
      }
      with open(f"{state_dir}/heartbeat.json", "w", encoding="utf-8") as f:
          json.dump(payload, f)
      print(json.dumps(payload), flush=True)
      time.sleep(10)
''
```

### Case 2: NixOS module with strict `systemd` sandboxing

Wrap `edge-telemetry` in a NixOS module that enforces `DynamicUser=true`, `ProtectSystem="strict"`, `ProtectHome=true`, and `PrivateTmp=true`. The daemon gets write access to `/var/lib/private/edge-telemetry` via `StateDirectory` and sees a read-only root filesystem everywhere else.

Save as `modules/edge-telemetry.nix`:

```nix
# modules/edge-telemetry.nix
{ config, lib, pkgs, ... }:
let
  cfg = config.services.edge-telemetry;
  pkg = import ../pkgs/edge-telemetry.nix { inherit pkgs; };
in
{
  options.services.edge-telemetry.enable =
    lib.mkEnableOption "hardened edge telemetry daemon";

  config = lib.mkIf cfg.enable {
    systemd.services.edge-telemetry = {
      description = "Edge Telemetry Daemon";
      wantedBy = [ "multi-user.target" ];
      after = [ "network.target" ];

      serviceConfig = {
        ExecStart = "${pkg}/bin/edge-telemetry";
        Restart = "always";
        RestartSec = "5s";

        # Ephemeral UID/GID allocated on start, released on stop
        DynamicUser = true;
        StateDirectory = "edge-telemetry";

        # Filesystem & namespace hardening
        ProtectSystem = "strict";
        ProtectHome = true;
        PrivateTmp = true;
        PrivateDevices = true;
        ProtectKernelTunables = true;
        ProtectKernelModules = true;
        ProtectControlGroups = true;
        NoNewPrivileges = true;
        RestrictSUIDSGID = true;
        CapabilityBoundingSet = "";
      };
    };
  };
}
```

### Case 3: Anchor `system.configurationRevision` to `self.rev` and wire `deploy-rs`

In `flake.nix`, bind `system.configurationRevision` to `self.rev` (when the Git tree is clean) or `self.dirtyRev` (during local testing), and declare the `deploy.nodes.edge-01` target with `magicRollback = true`.

Save as `flake.nix`:

```nix
# flake.nix
{
  description = "Lab H2 — Git-SHA Edge VM with Magic Rollback";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-26.05";
    deploy-rs.url = "github:serokell/deploy-rs";
    deploy-rs.inputs.nixpkgs.follows = "nixpkgs";
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
      nixosConfigurations.edge-01 = nixpkgs.lib.nixosSystem {
        inherit system;
        modules = [
          ./modules/edge-telemetry.nix
          ({ ... }: {
            networking.hostName = "edge-01";
            system.stateVersion = "26.05";

            # Stamp the exact git commit SHA into the OS generation
            system.configurationRevision = self.rev or self.dirtyRev or "unknown";

            services.edge-telemetry.enable = true;
            services.openssh.enable = true;
            networking.firewall.enable = true;
            networking.firewall.allowedTCPPorts = [ 22 ];
          })
        ];
      };

      deploy.nodes.edge-01 = {
        hostname = "10.100.50.20";
        magicRollback = true;
        confirmTimeout = 30;
        profiles.system = {
          sshUser = "root";
          user = "root";
          path = deployPkgs.deploy-rs.lib.activate.nixos self.nixosConfigurations.edge-01;
        };
      };

      checks = builtins.mapAttrs
        (system: deployLib: deployLib.deployChecks self.deploy)
        deploy-rs.lib;
    };
}
```

Commit the clean tree, deploy `edge-01`, and verify the stamped Git SHA and systemd sandbox hardness score:

```bash
git add -A && git commit -m "edge-01: initial telemetry release"
nix run github:serokell/deploy-rs -- .#edge-01

ssh root@10.100.50.20 'nixos-version --configuration-revision'
ssh root@10.100.50.20 'systemd-analyze security edge-telemetry.service | tail -n 1'
```

Output shows the exact 40-character Git commit SHA and an `OK` / `SAFE` exposure rating from `systemd-analyze security`.

### Case 4: Verification — inject a broken SSH firewall commit and watch magic-rollback

Now simulate a catastrophic operator error: remove port `22` from `allowedTCPPorts` and explicitly disable `services.openssh.openFirewall` so activating the new profile drops all SSH traffic immediately.

Edit the host module inside `flake.nix`:

```nix
# Broken firewall change in flake.nix
services.openssh.enable = true;
services.openssh.openFirewall = false;
networking.firewall.enable = true;
networking.firewall.allowedTCPPorts = [ ]; # BUG: locked out port 22!
```

Commit the bad change and push it with `deploy-rs`:

```bash
git commit -am "edge-01: broken firewall rule"
nix run github:serokell/deploy-rs -- .#edge-01
```

Observe the rollout sequence:
1. `deploy-rs` uploads the new closure over SSH and activates the profile.
2. `nftables` reloads and drops TCP port 22.
3. `deploy-rs` attempts to connect to `10.100.50.20:22` to confirm activation and blocks until `confirmTimeout` (`30s`) expires.
4. On `edge-01`, the unconfirmed `deploy-rs` rollback canary fires after 30 seconds, switches the system profile back to the previous working generation, and reloads `nftables` with port 22 open.

Verify from your workstation after 35 seconds:

```bash
ssh root@10.100.50.20 'nixos-version --configuration-revision'
```

The revision reported by `edge-01` matches the **previous** healthy commit SHA, not the broken firewall commit.

## The trap

The trap is **setting `services.openssh.enable = true` and assuming removing `22` from `networking.firewall.allowedTCPPorts` will lock you out**. In NixOS, `services.openssh.openFirewall` defaults to `true`, which automatically appends `services.openssh.ports` (`[ 22 ]`) to the firewall even if `allowedTCPPorts = [ ]`. To actually test an SSH firewall lockout (or if you restrict SSH to a specific interface), you must set `services.openssh.openFirewall = false`.

The second trap is **deploying from a dirty Git tree in production**. When the working tree has uncommitted edits, `self.rev` is not set (only `self.dirtyRev` exists), so `nixos-version --configuration-revision` cannot prove which branch commit is running. Enforce `test -n "${self.rev or ""}"` or run deployments strictly from CI commits.

## The boring rule

- Always bind `system.configurationRevision = self.rev or self.dirtyRev or "unknown"` in your base host module.
- Lock every custom daemon with `DynamicUser = true`, `ProtectSystem = "strict"`, `ProtectHome = true`, and `PrivateTmp = true`.
- Keep `magicRollback = true` with a bounded `confirmTimeout` (`30`–`60` seconds) on all remote `deploy-rs` targets.
- Test the rollback path deliberately with `services.openssh.openFirewall = false` before shipping an edge box to the field.

## Try this

1. Run `systemd-analyze security edge-telemetry.service` on `edge-01` and inspect which directive eliminates each remaining exposure point.
2. Exec into the service namespace (`nsenter` or `ExecStartPre` test) and confirm the daemon cannot write to `/etc` or `/home` and sees an isolated `/tmp`.
3. Add a Nix assertion that fails evaluation in CI if `self ? dirtyRev` so production builds can only originate from clean Git commits.
