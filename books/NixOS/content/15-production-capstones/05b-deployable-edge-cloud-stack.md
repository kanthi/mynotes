---
title: 'Deployable Edge Cloud ("Verlegefähige Cloud"): Talos, Proxmox, and NixOS'
---

# Deployable Edge Cloud ("Verlegefähige Cloud"): Talos, Proxmox, and NixOS

A portable, ruggedized cloud rack ("verlegefähige Cloud") shipped to a field exercise ("Übungsvorhaben") or operated in a high-security public-sector network cannot call `cache.nixos.org`, `registry-1.docker.io`, or `registry.opentofu.org` when the uplink goes dark. The boring default for a deployable open-source edge cloud is: **three physical Proxmox + Ceph nodes hosting one NixOS Seed/Bastion VM that serves every cache, image, git repo, and time signal locally, plus a Talos Linux Kubernetes cluster provisioned end-to-end by OpenTofu and GitOps**.

## Mental model

In a deployable rack, every external internet dependency must be terminated and mirrored on a single, deterministic **NixOS Seed VM** (`seed-01`) inside the rack. When the rack rolls out at a new site or powers up cold during an exercise, `seed-01` boots first from local storage and provides the entire supply chain to the rest of the stack.

```text
┌────────────────────────────────────────────────────────────────────────────┐
│ Portable 3-Node Rack: Proxmox VE + Ceph (pve-01, pve-02, pve-03)           │
│                                                                            │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ NixOS Seed & Bastion VM (seed-01 · 10.100.10.30)                     │  │
│  │  • Harmonia / nix-serve   (:5000)  ← nix flake archive + closures    │  │
│  │  • Zot OCI Registry       (:5001)  ← skopeo sync / dockerTools tars  │  │
│  │  • Forgejo Git Server     (:3000)  ← flake + generated Flux YAML     │  │
│  │  • Chrony NTP + CoreDNS   (:123/:53)                                 │  │
│  │  • OpenTofu + talosctl + Colmena operator devShell                   │  │
│  └──────────────┬───────────────────────────────────────┬───────────────┘  │
│                 │ OpenTofu (bpg/proxmox)                │ Flux GitOps +    │
│                 ▼ + talosctl (TCP 50000)                ▼ OCI mirror       │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ Talos Linux Kubernetes Cluster (cp-01..03, wk-01..03)                │  │
│  │  • Ceph-CSI RBD PVCs backed by Proxmox Ceph (ceph-k8s pool)          │  │
│  │  • Pulls all container images from https://seed-01.rack.internal:5001│  │
│  │  • Reconciles workloads from https://seed-01.rack.internal:3000      │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────────┘
```

| Phase | Action | Network Required |
|---|---|---|
| **1. Home Base Prep** | Build NixOS closures, OCI images, and Helm YAML; pack onto `seed-01` or transport disk | Connected (staging network) |
| **2. Field Cold Start** | Power on Proxmox nodes → verify Ceph quorum → boot `seed-01` | **Air-gapped / Disconnected** |
| **3. Automated Rollout** | From `seed-01`, run `tofu apply` + `talosctl` bootstrap against local mirrors | **Air-gapped / Disconnected** |
| **4. App Delivery** | Flux pulls manifests from `seed-01` Forgejo; kubelet pulls images from `seed-01` Zot | **Air-gapped / Disconnected** |

## Worked examples

### Case 1: The NixOS Seed VM (`seed-01`) module

`seed-01` combines an offline Nix binary cache ([Harmonia](../10-cicd/01-binary-caches.md)), an OCI container registry (`services.zot`), a local Git server (`services.forgejo`), and an authoritative local stratum-10 NTP server (`services.chrony`) so TLS certificates and `etcd` leases never fail when GPS or internet NTP is absent.

Save as `hosts/seed-01.nix`:

```nix
# hosts/seed-01.nix
{ config, pkgs, ... }:
{
  imports = [ ../modules/proxmox-guest.nix ];

  networking.hostName = "seed-01";
  system.stateVersion = "26.05";

  # 1. Local NTP for disconnected rack (Talos & Proxmox sync here)
  services.chrony = {
    enable = true;
    extraConfig = ''
      allow 10.100.0.0/16
      local stratum 10
    '';
  };

  # 2. Local Nix binary cache (signed with offline cache key)
  sops.secrets."harmonia-sign-key" = { mode = "0400"; };
  services.harmonia = {
    enable = true;
    signKeyPaths = [ config.sops.secrets."harmonia-sign-key".path ];
    settings.bind = "0.0.0.0:5000";
  };

  # 3. Local OCI registry for Talos system images & app workloads
  services.zot = {
    enable = true;
    settings = {
      storage.rootDirectory = "/var/lib/zot";
      http = {
        address = "0.0.0.0";
        port = "5001";
      };
    };
  };

  # 4. Local Git server for GitOps (Flux / Argo CD) and flake repo
  services.forgejo = {
    enable = true;
    settings.server = {
      HTTP_PORT = 3000;
      DOMAIN = "seed-01.rack.internal";
      ROOT_URL = "http://seed-01.rack.internal:3000/";
    };
  };

  networking.firewall.allowedTCPPorts = [ 22 3000 5000 5001 ];
  networking.firewall.allowedUDPPorts = [ 123 ];
}
```

### Case 2: Packing the offline release bundle before transport

Before the rack leaves the staging facility (or when preparing an encrypted transport SSD for an air-gapped site), populate `seed-01`'s Nix store and OCI registry with the exact closure of your flake, OpenTofu providers, and container images.

```bash
# 1. Archive all flake inputs and build the full rack closure
nix flake archive --to ssh://ops@10.100.10.30
nix copy --to ssh://ops@10.100.10.30 \
  .#nixosConfigurations.seed-01.config.system.build.toplevel \
  .#devShells.x86_64-linux.default \
  .#manifests

# 2. Push Nix-built container images directly into seed-01's Zot registry
nix build .#desk-api-image
skopeo copy --dest-tls-verify=false \
  docker-archive:./result \
  docker://10.100.10.30:5001/desk/desk-api:26.05

# 3. Mirror Talos & Kubernetes control-plane images listed by talosctl
nix develop --command talosctl image default | while read -r img; do
  target="10.100.10.30:5001/${img#*/}"
  skopeo copy --dest-tls-verify=false "docker://$img" "docker://$target"
done
```

Once `nix copy` and `skopeo copy` finish, unplug the rack's WAN uplink and run `nix develop --offline` on `seed-01` to prove no hidden network calls remain before shipping.

### Case 3: Pointing Talos at the NixOS Seed VM

In `talconfig.yaml` (from the [Talos chapter](../11-containers-and-kubernetes/06-talos-linux-with-nix.md)), patch every Talos node to use `seed-01` (`10.100.10.30`) for both NTP and all container image registries:

```yaml
# patches section in talconfig.yaml
patches:
  - |-
    machine:
      time:
        servers:
          - 10.100.10.30
      registries:
        mirrors:
          docker.io:
            endpoints: ["http://10.100.10.30:5001"]
          registry.k8s.io:
            endpoints: ["http://10.100.10.30:5001"]
          ghcr.io:
            endpoints: ["http://10.100.10.30:5001"]
          gcr.io:
            endpoints: ["http://10.100.10.30:5001"]
```

Without `local stratum 10` on `seed-01` and this `machine.time.servers` patch, Talos nodes waiting for internet NTP (`time.cloudflare.com`) will stall during boot and refuse to complete the bootstrap sequence in an air-gapped tent or bunker.

### Case 4: Zero-touch field rollout script

On site, an operator connects to `seed-01` and runs a single deterministic rollout script from the local git checkout:

```bash
#!/usr/bin/env bash
# scripts/rollout-rack.sh
set -euo pipefail

echo "[1/5] Checking Proxmox Ceph storage quorum..."
ssh root@10.100.20.11 'ceph health | grep -q HEALTH_OK'

echo "[2/5] Provisioning VMs on Proxmox via OpenTofu (offline providers)..."
nix develop --offline --command bash -c '
  cd tofu/proxmox
  tofu init -input=false
  tofu apply -input=false -auto-approve
'

echo "[3/5] Applying Talos machine configs and bootstrapping etcd..."
nix develop --offline --command bash -c '
  talhelper genconfig
  for ip in 10.100.10.11 10.100.10.12 10.100.10.13 10.100.10.21; do
    host=$(jq -r --arg ip "$ip" ".nodes[] | select(.ipAddress==\$ip) | .hostname" talconfig.yaml)
    talosctl apply-config --insecure --nodes "$ip" --file "clusterconfig/desk-edge-${host}.yaml"
  done
  talosctl bootstrap --nodes 10.100.10.11 --endpoints 10.100.10.11
  talosctl health --nodes 10.100.10.11 --wait-timeout 10m
  talosctl kubeconfig ./clusterconfig/kubeconfig --nodes 10.100.10.11 --endpoints 10.100.10.11
'

echo "[4/5] Applying Nix-generated CNI, Ceph-CSI, and Flux GitOps manifests..."
nix develop --offline --command kubectl apply -f ./generated/bootstrap/

echo "[5/5] Verifying cluster and application rollout..."
nix develop --offline --command kubectl get nodes,pods -A
```

### Case 5: Field exercise ("Übungsvorhaben") drill and training runbook

A deployable cloud is only as good as the operators running it under stress without the original author on the call. Keep a 1-page `docs/field-runbook.md` inside the repository and execute these four drills before signing off any release:

| Exercise Drill | Action | Pass Criterion |
|---|---|---|
| **1. WAN Unplug Cold Boot** | Pull the uplink cable; cold-boot all 3 Proxmox nodes | `seed-01` chrony serves NTP; Ceph reaches `HEALTH_OK`; Talos K8s API and pods recover in `< 8 min` |
| **2. Worker Node Loss** | Hard-power-off `pve-03` during active app traffic | Ceph RBD PVCs fail over cleanly; pods reschedule onto `wk-01`/`wk-02` within 5 minutes |
| **3. Node Re-Provision** | Destroy `wk-01` VM in Proxmox; re-run `scripts/rollout-rack.sh` | OpenTofu recreates VM; Talos pulls images from `seed-01:5001` and rejoins with zero internet traffic |
| **4. Operator Handover** | Hand a second engineer the locked hardware + `docs/field-runbook.md` + offline age key | Engineer completes a full cluster rebuild and app rollout without external help |

## The trap

The trap is **testing your "offline" rollout while the lab VLAN still has a default route to the internet**. Everything looks green during staging because one missed container image (`pause:3.10`, a `ceph-csi` sidecar, or an init container) silently falls back to Docker Hub, and `tofu init` silently fetches a provider. Then the rack arrives at the exercise site without internet and the bootstrap hangs at `ImagePullBackOff`. Always test the full wipe-and-rollout with the upstream gateway physically disconnected or dropped at the firewall.

The second trap is **forgetting NTP in an air-gapped environment**. Without `local stratum 10` on `seed-01`, `chrony` refuses to serve time when its own upstream servers are unreachable, causing Talos nodes to block boot indefinitely waiting for clock synchronization.

## The boring rule

- Every rack has a NixOS `seed-01` VM serving local NTP (`chrony` stratum 10), Nix binary cache (`harmonia`), OCI registry (`zot`), and Git (`forgejo`).
- Populate `seed-01` before transport with `nix flake archive`, `nix copy`, and `skopeo copy` for every output of `talosctl image default` plus your application images.
- Always run staging acceptance tests with the WAN uplink disabled and `nix develop --offline`.
- Keep the field exercise runbook (`docs/field-runbook.md`) in the same flake repository and verify it with a second operator.

## Try this

1. Run `talosctl image default` inside the `devShell` and compare that list against the images mirrored in your local registry script.
2. Disconnect network access in a test shell (`unshare -rn` or firewall drop) and verify `nix develop --offline --command tofu init -input=false` succeeds using only `/nix/store`.
3. Verify `chronyc sources` and `chronyc tracking` on a disconnected NixOS VM configured with `local stratum 10`.
4. Write the four-step cold-shutdown checklist for a 3-node Proxmox + Ceph + Talos rack (drain workers → `ceph osd set noout` → shutdown guests → power off hypervisors).
