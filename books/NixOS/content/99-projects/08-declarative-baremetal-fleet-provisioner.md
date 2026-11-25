---
title: '08. Project: Declarative Bare-Metal Fleet Provisioner'
---

# Project: Declarative Bare-Metal Fleet Provisioner

Build a reproducible bare-metal & VM fleet provisioner using **NixOS Flakes** (with iPXE/cloud-init or `nixos-anywhere`) targeting a multi-node Incus/QEMU lab—enforcing `systemd` service sandboxing, **SSH CA short-lived certificate authentication** (no static `~/.ssh/authorized_keys`), `node_exporter`, and automated post-boot hardware/kernel validation checks.

## Architecture

```text
Git Flake Repo (hosts/node-{1..3}.nix)
        |
        v  nixos-anywhere / iPXE netboot (zero manual SSH steps)
Target Nodes (Incus / QEMU / Bare Metal)
   ├── Immutable Nix store + declarative disko partitioning
   ├── Hardened systemd units (DynamicUser, ProtectSystem=strict, NoNewPrivileges)
   ├── OpenSSH TrustedUserCAKeys (short-lived SSH certs via jump host)
   └── Post-boot validator (kernel params, NUMA/NVMe checks, node_exporter :9100)
```

## Hardened NixOS Host Module Snippet

```nix
{ config, pkgs, ... }:
{
  services.openssh = {
    enable = true;
    settings = {
      PasswordAuthentication = false;
      KbdInteractiveAuthentication = false;
      PermitRootLogin = "no";
      TrustedUserCAKeys = "/etc/ssh/user_ca.pub";
    };
  };

  services.prometheus.exporters.node = {
    enable = true;
    enabledCollectors = [ "systemd" "processes" "tcpstat" ];
  };

  systemd.services.fleet-agent.serviceConfig = {
    DynamicUser = true;
    ProtectSystem = "strict";
    ProtectHome = true;
    PrivateTmp = true;
    NoNewPrivileges = true;
    RestrictAddressFamilies = [ "AF_INET" "AF_INET6" "AF_UNIX" ];
  };
}
```

## Acceptance Drills

- Wipe and reprovision `node-2` from scratch with a single command and verify it rejoins Prometheus scraping within 3 minutes.
- Attempt SSH login with an expired SSH certificate or static key and confirm rejection.
- Run `systemd-analyze security fleet-agent.service` and confirm an exposure score `< 2.0` (SAFE).
