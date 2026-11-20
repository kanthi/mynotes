---
title: Ceph Storage with Proxmox, NixOS, and Kubernetes
---

# Ceph Storage with Proxmox, NixOS, and Kubernetes

Shared storage is what turns three physical servers in a rack into a cluster that survives a dead node. In a Proxmox + Talos + NixOS architecture, **Proxmox VE manages the physical Ceph OSDs and monitors on bare metal, while NixOS hosts and Talos Kubernetes workloads consume that Ceph cluster as authenticated RBD and CephFS clients**.

## Mental model

Running a Ceph storage cluster inside disposable Kubernetes pods (Rook) on top of virtual disks that already live on Proxmox is double-virtualized storage overhead. When Proxmox already runs hyperconverged Ceph across your physical nodes (`pve-01`, `pve-02`, `pve-03`), keep the OSDs on the bare-metal hypervisor and expose dedicated pools to NixOS and Kubernetes:

```text
Physical Proxmox + Ceph Cluster (pve-01..03, 10.100.20.0/24 storage net)
    ├── Pool: ceph-vm     ──►  VM boot disks (NixOS & Talos root disks)
    ├── Pool: ceph-k8s    ──►  Talos Kubernetes PVCs via ceph-csi-rbd
    └── FS:   cephfs-data ──►  NixOS hosts (mount -t ceph) & K8s RWX volumes
```

| Consumer | Mechanism | Auth Secret Location |
|---|---|---|
| Proxmox Hypervisor | Native `pveceph` (`krbd` / `librbd`) | `/etc/pve/priv/ceph.*` |
| NixOS Host (`ops-01`, `seed-01`) | Kernel `ceph` / `rbd` module + `fileSystems` | `sops-nix` → `/run/secrets/ceph-client-nixos` |
| Talos Kubernetes Cluster | `ceph-csi` Helm chart generated from Nix | Encrypted Kubernetes `Secret` (SOPS / KSOPS) |

## Worked examples

### Case 1: Scoped Ceph client keyrings on the cluster

Never hand `client.admin` to a NixOS VM or a Kubernetes cluster. A compromised guest with `client.admin` can delete every VM disk pool on the hypervisor. Create least-privilege Ceph users scoped to their specific pools:

```bash
# Run once on pve-01 to create a scoped key for Kubernetes PVCs (ceph-k8s pool)
ceph osd pool create ceph-k8s 64
ceph osd pool application enable ceph-k8s rbd
ceph auth get-or-create client.k8s-csi \
  mon 'profile rbd' \
  osd 'profile rbd pool=ceph-k8s' \
  mgr 'profile rbd pool=ceph-k8s'

# Create a scoped CephFS user for NixOS artifact/cache sharing
ceph fs authorize cephfs client.nixos-seed / rw
```

Store the resulting key secrets in `secrets/ceph.sops.yaml`, never in plaintext `.nix` files.

### Case 2: Declarative CephFS mount on a NixOS host

When a NixOS host (such as a binary cache server or backup mover) needs direct access to a CephFS volume, enable the Ceph client tools, decrypt the secret via `sops-nix`, and declare the mount with `_netdev` so `systemd` waits for the storage network before mounting.

Save as `modules/ceph-client.nix`:

```nix
# modules/ceph-client.nix
{ config, pkgs, ... }:
{
  environment.systemPackages = [ pkgs.ceph-client ];

  sops.secrets."ceph-nixos-seed-secret" = {
    mode = "0400";
    owner = "root";
  };

  environment.etc."ceph/ceph.conf".text = ''
    [global]
    fsid = 4b5c8c0a-7e3a-4f12-9a11-001122334455
    mon_host = [v2:10.100.20.11:3300/0,v1:10.100.20.11:6789/0] [v2:10.100.20.12:3300/0,v1:10.100.20.12:6789/0] [v2:10.100.20.13:3300/0,v1:10.100.20.13:6789/0]
  '';

  fileSystems."/srv/shared-artifacts" = {
    device = "10.100.20.11,10.100.20.12,10.100.20.13:/";
    fsType = "ceph";
    options = [
      "name=nixos-seed"
      "secretfile=${config.sops.secrets."ceph-nixos-seed-secret".path}"
      "mds_namespace=cephfs"
      "_netdev"
      "noatime"
      "nofail"
    ];
  };
}
```

Include `"nofail"` alongside `"_netdev"`. If the storage switch powers up 20 seconds slower than the VM during a cold rack start, `"nofail"` ensures `sshd` still comes up so you can reach the host.

### Case 3: Rendering `ceph-csi-rbd` from Nix for Talos Kubernetes

Talos Linux includes the `rbd` and `nbd` kernel modules out of the box, so it requires no host package installation to mount Ceph RBD PersistentVolumeClaims. Following the [Helm Charts from Nix chapter](../11-containers-and-kubernetes/03-helm-charts-from-nix.md), render the `ceph-csi-rbd` manifests in Nix so GitOps applies plain, reviewable YAML.

Save the CSI values fragment as `k8s/ceph-csi-values.nix`:

```nix
# k8s/ceph-csi-values.nix
{
  csiConfig = [
    {
      clusterID = "4b5c8c0a-7e3a-4f12-9a11-001122334455";
      monitors = [
        "10.100.20.11:6789"
        "10.100.20.12:6789"
        "10.100.20.13:6789"
      ];
    }
  ];

  storageClass = {
    create = true;
    name = "ceph-rbd";
    clusterID = "4b5c8c0a-7e3a-4f12-9a11-001122334455";
    pool = "ceph-k8s";
    provisionerSecret = "csi-rbd-secret";
    controllerExpandSecret = "csi-rbd-secret";
    nodeStageSecret = "csi-rbd-secret";
    reclaimPolicy = "Delete";
    allowVolumeExpansion = true;
  };
}
```

 Note that `Talos` uses `/var/lib/kubelet` as its standard kubelet root directory, so standard `ceph-csi` daemonset mounts work without path overrides on modern Talos releases (`v1.7+`).

### Case 4: Cold-boot and quorum discipline on a 3-node rack

In a 3-node hyperconverged cluster (`size = 3`, `min_size = 2`), Ceph requires **at least 2 of 3 monitors (`mon`) and 2 of 3 OSD hosts** online before any I/O is allowed.

Before powering down a portable rack or performing maintenance on a node, verify health and suppress unnecessary rebalancing from your NixOS ops shell:

```bash
# Before planned single-node reboot or rack shutdown:
ssh root@10.100.20.11 'ceph status && ceph osd set noout'

# After all 3 nodes return online:
ssh root@10.100.20.11 'ceph osd unset noout && ceph health detail'
```

`HEALTH_OK` with all PGs `active+clean` is the gate before starting heavy database deployments.

## The trap

The trap is **omitting `_netdev` and `nofail` on a NixOS Ceph mount**. Without them, `systemd` treats `/srv/shared-artifacts` as a local block device required for `local-fs.target`. If a Ceph monitor is unreachable at boot, the NixOS guest drops into emergency mode on the console with `sshd` dead.

The second trap is **setting pool `min_size = 1` on a 3-node rack "so it still works with two nodes down"**. `min_size = 1` allows split-brain writes and silent data loss when an OSD flaps during field power transitions. Keep `size = 3, min_size = 2` and fix the dead node.

## The boring rule

- Keep Ceph OSDs on the bare-metal hypervisor (Proxmox); do not layer Rook-Ceph on top of Ceph-backed virtual disks.
- Scope Ceph authentication per consumer (`client.k8s-csi` for `ceph-k8s`, `client.nixos-seed` for `cephfs`). Never distribute `client.admin`.
- Always pass `_netdev` and `nofail` on NixOS `fileSystems` mounts backed by Ceph.
- Render `ceph-csi` manifests in CI from Nix and deliver the `csi-rbd-secret` through SOPS/KSOPS.
- Set `ceph osd set noout` before planned node reboots or rack power-downs.

## Try this

1. Evaluate `modules/ceph-client.nix` in a test configuration and inspect the generated `/etc/fstab` entry (`cat result/etc/fstab`) to verify `_netdev,nofail` and the `/run/secrets/...` path.
2. Verify why `secretfile=/run/secrets/...` is safe while `secret=${builtins.readFile ...}` would leak the Ceph key into `/nix/store`.
3. Write a pre-flight shell check that fails if `ceph health` returns `HEALTH_ERR` or if `noout` was accidentally left set after a maintenance window.
4. Trace the storage path of a PostgreSQL pod running on Talos: Pod PVC → `ceph-csi` kernel RBD device on the Talos worker → `10.100.20.0/24` storage VLAN → Proxmox bare-metal NVMe OSDs.
