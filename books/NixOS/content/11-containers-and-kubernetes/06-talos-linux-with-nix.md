---
title: Talos Linux from a Nix Flake
---

# Talos Linux from a Nix Flake

Running full NixOS inside every disposable Kubernetes worker means maintaining two declarative control planes on the same node: `nixos-rebuild` for the OS and the kubelet for the pods. The boring default when you want a dedicated Kubernetes cluster alongside NixOS infrastructure is: **NixOS owns the stateful support hosts (bastion, binary cache, GitOps seed) and a flake `devShell` drives Talos Linux for the immutable, API-only Kubernetes nodes**.

## Mental model

Talos Linux has no SSH, no shell, and no package manager. Its entire state is a single machine configuration YAML applied over mTLS via `talosctl`. Nix complements Talos by pinning the exact client toolchain (`talosctl`, `talhelper`, `kubectl`, `cilium-cli`) and generating or validating those machine configs deterministically from git alongside your `sops` secrets.

```text
flake.nix (devShell: talosctl, talhelper, sops, kubectl)
    │
    ├── talconfig.yaml + secrets.sops.yaml
    │       └── talhelper genconfig  →  clusterconfig/*.yaml
    │
    ├── NixOS hosts (ops-01 / seed)  ── Colmena / nixos-rebuild
    └── Talos nodes (cp-01..03, wk)  ── talosctl apply-config (mTLS, port 50000)
```

| Concern | NixOS Host (`ops-01`) | Talos Node (`cp-01`, `wk-01`) |
|---|---|---|
| Management transport | SSH (port 22) + `nixos-rebuild` / Colmena | Talos API (TCP 50000, mTLS) + `talosctl` |
| Interactive shell | `bash` / `zsh` for operators | None (`talosctl dmesg`, `talosctl logs`, debug pods) |
| Configuration unit | NixOS module graph → `/nix/store` generation | MachineConfig YAML → A/B SquashFS image |
| Secrets | `sops-nix` → `/run/secrets/*` | `talhelper` + SOPS → cluster PKI & bootstrap tokens |

Do not try to force `nixos-rebuild` onto Talos, and do not run three different versions of `talosctl` across operator laptops. The flake `devShell` is the single control surface.

## Worked examples

### Case 1: Hermetic Talos and Kubernetes operator shell

Save as `flake.nix`:

```nix
# flake.nix
{
  description = "Desk Talos + NixOS control plane";

  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-26.05";

  outputs = { self, nixpkgs }:
    let
      pkgs = nixpkgs.legacyPackages.x86_64-linux;
    in
    {
      devShells.x86_64-linux.default = pkgs.mkShell {
        packages = [
          pkgs.talosctl
          pkgs.talhelper
          pkgs.kubectl
          pkgs.kubernetes-helm
          pkgs.cilium-cli
          pkgs.sops
          pkgs.age
          pkgs.jq
        ];
        shellHook = ''
          export TALOSCONFIG="$PWD/clusterconfig/talosconfig"
          export KUBECONFIG="$PWD/clusterconfig/kubeconfig"
        '';
      };
    };
}
```

Verify the pinned binaries come from `/nix/store`:

```bash
nix develop --command bash -c 'which talosctl talhelper kubectl; talosctl version --client'
```

Setting `TALOSCONFIG` and `KUBECONFIG` inside `shellHook` prevents operators from accidentally firing `talosctl reset` or `kubectl delete` against whatever cluster was left in `~/.talos/config` or `~/.kube/config`.

### Case 2: Declarative cluster inventory with `talhelper` and SOPS

Raw Talos `controlplane.yaml` and `worker.yaml` files repeat certificates and machine tokens across every node. `talhelper` separates topology (`talconfig.yaml`, committed in plaintext) from PKI material (`talsecret.sops.yaml`, encrypted with SOPS).

Generate and encrypt the cluster secrets once inside the shell:

```bash
nix develop --command bash -c '
  talhelper gensecret > talsecret.sops.yaml
  sops --encrypt --in-place talsecret.sops.yaml
'
```

Save the cluster topology as `talconfig.yaml`:

```yaml
# talconfig.yaml
clusterName: desk-edge
talosVersion: v1.9.4
kubernetesVersion: v1.32.2
endpoint: https://10.100.10.10:6443
cniConfig:
  name: custom
nodes:
  - hostname: cp-01
    ipAddress: 10.100.10.11
    installDisk: /dev/sda
    controlPlane: true
  - hostname: cp-02
    ipAddress: 10.100.10.12
    installDisk: /dev/sda
    controlPlane: true
  - hostname: cp-03
    ipAddress: 10.100.10.13
    installDisk: /dev/sda
    controlPlane: true
  - hostname: wk-01
    ipAddress: 10.100.10.21
    installDisk: /dev/sda
    controlPlane: false
```

Render the per-node configurations:

```bash
nix develop --command talhelper genconfig
```

`clusterconfig/` now holds `desk-edge-cp-01.yaml` through `wk-01.yaml` plus `talosconfig`. Add `clusterconfig/` to `.gitignore` immediately because the rendered files contain decrypted private keys.

### Case 3: Pure Nix validation check for Talos patches

If you prefer generating or patching Talos machine configs from Nix modules (for example, sharing NTP servers, registry mirrors, and VLAN IDs with your NixOS hosts), wrap `talosctl validate` in a flake check so invalid machine configs never merge.

Save as `checks-talos.nix` (or inline in `flake.nix`):

```nix
# fragment for outputs.checks.x86_64-linux
let
  basePatch = builtins.toJSON {
    machine = {
      time.servers = [ "10.100.0.30" ];
      registries.mirrors."docker.io".endpoints = [ "https://registry.desk.internal" ];
    };
  };
in
{
  checks.x86_64-linux.talos-patch-json = pkgs.runCommand "check-talos-patch" {
    nativeBuildInputs = [ pkgs.jq ];
  } ''
    echo '${basePatch}' | jq -e '.machine.time.servers[0] == "10.100.0.30"' > "$out"
  '';
}
```

Run `nix flake check` in CI. Your Talos registry mirrors and NTP IP now come from the exact same Nix constants that configure `services.chrony` and the OCI registry on your NixOS bastion.

### Case 4: Bootstrap without SSH

Boot the VMs from the Talos ISO (or Factory schematic qcow2 image). While sitting in maintenance mode, apply the rendered configurations and bootstrap `etcd` on the first control-plane node:

```bash
# First apply against maintenance mode uses --insecure
talosctl apply-config --insecure --nodes 10.100.10.11 --file clusterconfig/desk-edge-cp-01.yaml
talosctl apply-config --insecure --nodes 10.100.10.12 --file clusterconfig/desk-edge-cp-02.yaml
talosctl apply-config --insecure --nodes 10.100.10.13 --file clusterconfig/desk-edge-cp-03.yaml
talosctl apply-config --insecure --nodes 10.100.10.21 --file clusterconfig/desk-edge-wk-01.yaml

# Bootstrap etcd ONCE on a single control plane node
talosctl bootstrap --nodes 10.100.10.11 --endpoints 10.100.10.11

# Fetch kubeconfig into clusterconfig/kubeconfig
talosctl kubeconfig ./clusterconfig/kubeconfig --nodes 10.100.10.11 --endpoints 10.100.10.11
```

Once `etcd` forms quorum and Cilium is installed, Flux or Argo CD takes over from the [GitOps chapter](04-gitops-argo-flux.md).

### Case 5: Atomic upgrades and deterministic wipe

Upgrading Talos OS or wiping a node back to clean disk is a single API call from the `devShell`:

```bash
# Check current OS and kernel
talosctl version --nodes 10.100.10.21

# Atomic A/B image upgrade (rolls back automatically if boot fails)
talosctl upgrade --nodes 10.100.10.21 \
  --image factory.talos.dev/installer/376567988ad370138ad8b2698212367b8edcb69b5fd68c80be1f2ec7d603b4ba:v1.9.4

# Decommission and wipe disk cleanly
talosctl reset --nodes 10.100.10.21 --graceful=true --reboot
```

## The trap

The trap is **committing the rendered `clusterconfig/*.yaml` or `talosconfig` to git**. Those files contain the root CA private key for the Talos API and Kubernetes `etcd`. Anyone with `talosconfig` has root-equivalent control over every node on TCP 50000. Commit only `talconfig.yaml` and `talsecret.sops.yaml`.

The second trap is **running `talosctl bootstrap` on all three control-plane nodes**. That forms three separate single-node `etcd` clusters that never join. Run `talosctl bootstrap` exactly once against `cp-01`; `cp-02` and `cp-03` join automatically because they share the rendered config.

## The boring rule

- Pin `talosctl`, `talhelper`, and `kubectl` in the repo's `flake.nix`.
- Scope `TALOSCONFIG` and `KUBECONFIG` to the project directory in `shellHook`.
- Commit `talconfig.yaml` + `talsecret.sops.yaml`; gitignore `clusterconfig/`.
- Share infrastructure constants (NTP, DNS, OCI mirror URLs) between NixOS modules and Talos machine patches.
- Bootstrap `etcd` once on `cp-01`; hand workloads immediately to Flux or Argo CD.

## Try this

1. Enter the `devShell` and confirm `echo $TALOSCONFIG` points inside the repository, not `~/.talos/config`.
2. Run `talhelper gensecret` in a scratch folder, inspect the keys it creates, and verify `.gitignore` excludes `clusterconfig/`.
3. Add a machine patch in `talconfig.yaml` that points containerd at an internal NixOS registry mirror and validate the rendered YAML with `talosctl validate --config clusterconfig/desk-edge-wk-01.yaml --mode metal`.
4. List the firewall ports required between your NixOS management host and a Talos control plane (`50000/tcp` for Talos API, `6443/tcp` for Kubernetes API).
