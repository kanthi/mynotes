---
title: Proxmox VE Templates and OpenTofu Rollout
---

# Proxmox VE Templates and OpenTofu Rollout

Proxmox VE is a rock-solid hypervisor in public-sector, edge, and homelab racks, but its web UI invites click-ops drift. The boring default when your target hypervisor is Proxmox is: **build your NixOS guest template with `nixos-generators`, pin the `bpg/proxmox` provider in `pkgs.opentofu.withPlugins`, and provision both NixOS and Talos VMs through `tofu apply` without touching the browser**.

## Mental model

You do not need to replace Proxmox on the bare metal when the customer or hardware contract mandates Proxmox VE + Ceph. Instead, treat Proxmox purely as an API endpoint (port `8006/tcp`). Nix builds deterministic VM artifacts (`.vma.zst` or `.qcow2`) and provides an offline-capable OpenTofu wrapper that clones or boots those artifacts into running guests.

```text
nixos-generators (format = "proxmox")
    │
    ▼
nixos-template.vma.zst  ──►  Proxmox Storage (local-lvm / ceph-rbd)
                                 │
nix develop (tofu + bpg/proxmox) │
    │                            ▼
    └──► tofu apply  ──►  Clone NixOS VM (ID 9000 → 101) + Cloud-Init
                     ──►  Boot Talos VM (ID 201..204) from ISO/qcow2
```

| Guest Type on Proxmox | Image Source | Day-1 Bootstrap | Day-2 Management |
|---|---|---|---|
| **NixOS Service / Seed VM** | `nixos-generators` (`proxmox` VMA or `qcow2`) | Cloud-Init injects SSH key + static IP | Colmena / `deploy-rs` over SSH |
| **Talos Kubernetes Node** | Talos Factory ISO or `nocloud` qcow2 | `talosctl apply-config --insecure` | `talosctl` over TCP 50000 |

## Worked examples

### Case 1: Building a NixOS Proxmox template in a flake

A NixOS guest on Proxmox needs `qemu-guest-agent` so the Proxmox API can read its IP interfaces and execute clean shutdowns, plus `cloud-init` (or static kernel networking) so OpenTofu can assign per-clone addresses.

Save as `modules/proxmox-guest.nix`:

```nix
# modules/proxmox-guest.nix
{ modulesPath, ... }:
{
  imports = [
    "${modulesPath}/virtualisation/proxmox-image.nix"
  ];

  system.stateVersion = "26.05";
  services.qemuGuest.enable = true;
  services.cloud-init = {
    enable = true;
    network.enable = true;
  };

  services.openssh = {
    enable = true;
    settings.PasswordAuthentication = false;
  };

  networking.firewall.enable = true;
}
```

Expose the VMA archive as a flake package:

```nix
# fragment in flake.nix outputs
packages.x86_64-linux.proxmox-template = (nixpkgs.lib.nixosSystem {
  system = "x86_64-linux";
  modules = [ ./modules/proxmox-guest.nix ];
}).config.system.build.VMA;
```

Build the archive and restore it once as template ID `9000` on a Proxmox node:

```bash
nix build .#proxmox-template
ls -lh result/vzdump-qemu-*.vma.zst

# On the Proxmox host (or via automated upload):
qmrestore vzdump-qemu-nixos-26.05.vma.zst 9000 --storage ceph-vm
qm template 9000
```

### Case 2: OpenTofu with the pinned `bpg/proxmox` provider

Consistent with the [OpenTofu chapter](02-opentofu-integration.md), wrap `pkgs.opentofu` with `withPlugins` so `tofu init` never hits the internet on an air-gapped or restricted network.

Save in `flake.nix`:

```nix
# devShell fragment in flake.nix
let
  pkgs = nixpkgs.legacyPackages.x86_64-linux;
  tofu = pkgs.opentofu.withPlugins (p: [
    p.bpg_proxmox
    p.random
    p.local
  ]);
in
{
  devShells.x86_64-linux.default = pkgs.mkShell {
    packages = [ tofu pkgs.sops pkgs.jq ];
  };
}
```

Verify offline initialization:

```bash
nix develop --command tofu init -input=false
```

### Case 3: Least-privilege Proxmox API token via SOPS

Never put `root@pam` passwords into OpenTofu variables. Create a dedicated role and API token on Proxmox (`terraform@pve!automation`) with `VM.Allocate`, `VM.Clone`, `VM.Config.*`, `VM.PowerMgmt`, and `Datastore.AllocateSpace`, then store the token in SOPS.

Export the token into the shell at apply time without touching disk:

```bash
eval "$(sops -d --output-type dotenv secrets/proxmox.sops.yaml | sed 's/^/export /')"
# Exports PROXMOX_VE_ENDPOINT="https://pve-01.desk.internal:8006/"
# and PROXMOX_VE_API_TOKEN="terraform@pve!automation=xxxxxxxx-xxxx-..."
nix develop --command tofu plan
```

### Case 4: Provisioning a NixOS bastion and Talos nodes together

Save as `tofu/proxmox/main.tf`:

```hcl
# tofu/proxmox/main.tf
terraform {
  required_providers {
    proxmox = {
      source = "bpg/proxmox"
    }
  }
}

provider "proxmox" {
  insecure = false
}

resource "proxmox_virtual_environment_vm" "nixos_ops" {
  name      = "ops-01"
  node_name = "pve-01"
  vm_id     = 101

  agent {
    enabled = true
  }

  clone {
    vm_id = 9000
    full  = true
  }

  cpu {
    cores = 4
    type  = "host"
  }

  memory {
    dedicated = 8192
  }

  network_device {
    bridge = "vmbr0"
  }

  initialization {
    ip_config {
      ipv4 {
        address = "10.100.10.30/24"
        gateway = "10.100.10.1"
      }
    }
    user_account {
      username = "ops"
      keys     = ["ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... ops@desk"]
    }
  }
}

resource "proxmox_virtual_environment_vm" "talos_cp" {
  count     = 3
  name      = "cp-0${count.index + 1}"
  node_name = "pve-0${count.index + 1}"
  vm_id     = 201 + count.index

  cpu {
    cores = 4
    type  = "host"
  }

  memory {
    dedicated = 8192
  }

  disk {
    datastore_id = "ceph-vm"
    interface    = "scsi0"
    size         = 64
    file_format  = "raw"
  }

  cdrom {
    file_id = "cephfs:iso/talos-v1.9.4-metal-amd64.iso"
  }

  network_device {
    bridge = "vmbr0"
  }
}
```

Notice `node_name = "pve-0${count.index + 1}"` on the Talos control-plane VMs: each control-plane node lands on a different physical Proxmox host, and their virtual disks live on the shared `ceph-vm` RBD pool. Once `ops-01` boots from template `9000`, Colmena immediately takes over its full NixOS configuration.

## The trap

The trap is **enabling `agent { enabled = true }` in OpenTofu while `services.qemuGuest.enable` is false in the VM**. OpenTofu will wait 15 minutes for the QEMU Guest Agent to report an IP address over the virtio-serial socket and then fail the apply. Always enable `services.qemuGuest.enable = true` in the NixOS template, and install the `siderolabs/qemu-guest-agent` system extension in your Talos image schematic if you enable the agent flag on Talos VMs.

The second trap is **leaving Cloud-Init enabled forever after Colmena takes over**. If Cloud-Init re-runs on a later boot with stale Proxmox metadata, it can overwrite `/etc/ssh/authorized_keys` or routes. Use Cloud-Init for the initial clone birth, or disable `services.cloud-init.network.enable` once your declarative NixOS networking module is active.

## The boring rule

- Build the NixOS Proxmox base template from the flake (`proxmox-image.nix`), never from a hand-installed ISO.
- Always set `services.qemuGuest.enable = true` on NixOS Proxmox guests.
- Pin `bpg_proxmox` via `pkgs.opentofu.withPlugins`.
- Authenticate with a scoped Proxmox API token decrypted from SOPS into environment variables (`PROXMOX_VE_API_TOKEN`), never `root@pam`.
- Spread `cp-01..03` across `pve-01..03` backed by shared Ceph storage.

## Try this

1. Inspect `nixpkgs/nixos/modules/virtualisation/proxmox-image.nix` to see what disk layout and serial console flags it sets by default.
2. Run `nix develop --command tofu providers` in the Proxmox OpenTofu directory and confirm `bpg/proxmox` resolves from `/nix/store`.
3. Write a `tofu plan` check that asserts no VM uses `root@pam` or plaintext passwords in `.tf` files.
4. Trace what happens when `ops-01` is destroyed and re-cloned: OpenTofu recreates VM `101`, Cloud-Init sets `10.100.10.30`, and `colmena apply --on ops-01` converges the host in under two minutes.
