---
title: Lab Host — Linux
---

# Lab Host — Linux

Prepare a **Linux** machine (bare metal, cloud VM, or local VM from [Lab Host — macOS](01-lab-host-macos.md)) as a Containerlab control host. Native Linux is the gold-standard lab platform: containers share the host kernel directly, `veth` pairs and Linux bridges run at full speed, and `tcpdump`/`ip netns` work without a virtualization boundary.

## Learning goals

By the end of this chapter you can:

- Install Docker Engine (or evaluate Podman) and Containerlab on Ubuntu/Debian or Fedora/RHEL
- Load and persist the kernel modules required for bridges, VLANs, VXLAN, WireGuard, and MPLS labs
- Tune `sysctl` parameters (`inotify`, `ip_forward`, `rp_filter`) so multi-node topologies do not stall
- Verify cgroups v2 and avoid `ufw`/`nftables`/`firewalld` forward-chain traps with Docker bridges
- Run a repeatable host health check and first smoke test

## Prerequisites

- 64-bit Linux (`x86_64` or `arm64`), kernel 5.15+ recommended (Ubuntu 22.04/24.04 LTS, Debian 12, or Fedora 40+)
- `sudo` privileges for package installation, kernel modules, and `containerlab deploy`
- **CPU/RAM:** 2 vCPUs / 4 GB RAM minimum for small FRR topologies; 4+ vCPUs / 16 GB RAM for Clos fabrics, SR Linux, or cEOS
- **Nested virtualization (`/dev/kvm`):** only required if you plan to run VM-based `vrnetlab` nodes; pure container labs (FRR, Alpine, SR Linux, cEOS) do **not** need KVM

Check hardware virtualization and kernel version:

```bash
uname -r
ls -l /dev/kvm 2>/dev/null || echo "No KVM (container-only labs are still fine)"
free -h
```

## Container runtime: Docker Engine vs Podman

Containerlab supports **Docker Engine** as its primary runtime and has experimental/partial support for **Podman**.

| Aspect | Docker Engine (Recommended) | Podman |
|---|---|---|
| Containerlab maturity | First-class; all kinds tested | Supported for basic kinds; edge cases with some NOS images |
| Networking & `clab` bridge | Seamless with `docker` bridge driver | Requires rootful `podman` for full `veth`/netns wiring |
| Daemon model | `dockerd` systemd service | Daemonless (or `podman.socket` for Docker API compatibility) |
| Book recommendation | **Use Docker Engine** for lowest friction | Only if your org mandates Podman and you test your target NOS |

### Install Docker Engine (Ubuntu / Debian)

Use Docker's official repository rather than the older `docker.io` distro package:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg iproute2 tcpdump jq bridge-utils

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker "$USER"
sudo systemctl enable --now docker
```

Log out and back in (or run `newgrp docker`), then verify:

```bash
docker version
docker info --format 'Server: {{.ServerVersion}} | Cgroup: {{.CgroupVersion}} | Driver: {{.Driver}}'
```

### Install Docker Engine (Fedora)

```bash
sudo dnf -y install dnf-plugins-core iproute tcpdump jq
sudo dnf-3 config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
sudo dnf -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker "$USER"
sudo systemctl enable --now docker
```

## Install Containerlab

Containerlab provides an official setup script (and `.deb`/`.rpm` repositories) at [containerlab.dev](https://containerlab.dev/install/):

```bash
curl -sL https://containerlab.dev/setup | sudo -E bash -s "all"
containerlab version
```

Optional shell completion (bash/zsh):

```bash
containerlab completion bash | sudo tee /etc/bash_completion.d/containerlab > /dev/null
```

## Required kernel modules

Because containers share the host Linux kernel, a container cannot create a VXLAN VTEP, 802.1Q VLAN subinterface, WireGuard tunnel, or MPLS label-switched path unless the **host kernel** has the corresponding module loaded.

| Module | Required For |
|---|---|
| `br_netfilter` | Linux bridge packet filtering and Docker bridge behavior |
| `8021q` | VLAN tagging (`eth1.10`) and trunk labs |
| `vxlan` | VXLAN tunnels and EVPN-VXLAN fabrics |
| `wireguard` | WireGuard VPN and MTU labs |
| `mpls_router`, `mpls_iptunnel` | MPLS / L3VPN / Segment Routing labs |
| `dummy` | Loopback and dummy test interfaces |
| `sch_netem` | Link impairment (latency, jitter, packet loss via `tc netem`) |

Load them immediately and persist across reboots:

```bash
sudo tee /etc/modules-load.d/containerlab.conf > /dev/null <<'EOF'
br_netfilter
8021q
vxlan
wireguard
mpls_router
mpls_iptunnel
dummy
sch_netem
EOF

for mod in br_netfilter 8021q vxlan wireguard mpls_router mpls_iptunnel dummy sch_netem; do
  sudo modprobe "$mod" || echo "WARN: module $mod not available in this kernel"
done
```

> [!NOTE]
> Minimal cloud kernels (such as `linux-image-cloud-*` or `linux-aws`) sometimes omit `mpls_router` or `sch_netem`. On Ubuntu, installing `linux-modules-extra-$(uname -r)` restores them.

## Sysctl tuning for multi-node labs

Large topologies (8+ FRR or SR Linux containers) quickly hit default Linux `inotify` limits, causing systemd/NOS daemons inside containers to hang or fail with `too many open files`. Routing and asymmetric-path labs also trip strict reverse-path filtering (`rp_filter`).

Create `/etc/sysctl.d/99-containerlab.conf`:

```ini
# Prevent "too many open files" / inotify exhaustion in multi-node labs
fs.inotify.max_user_instances = 8192
fs.inotify.max_user_watches = 524288

# Enable IPv4 and IPv6 forwarding on the lab host
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1

# Loose reverse-path filtering (2) avoids silent drops on asymmetric lab paths
net.ipv4.conf.all.rp_filter = 2
net.ipv4.conf.default.rp_filter = 2

# Raise ARP/NDP neighbor table thresholds for large L2/fabric labs
net.ipv4.neigh.default.gc_thresh1 = 1024
net.ipv4.neigh.default.gc_thresh2 = 4096
net.ipv4.neigh.default.gc_thresh3 = 8192
```

Apply immediately:

```bash
sudo sysctl --system
```

## Cgroups v2 check

Modern distros (Ubuntu 22.04+, Debian 12+, Fedora 31+) default to **cgroups v2** (`unified` hierarchy), which is required by recent Nokia SR Linux and Arista cEOS releases. Verify cgroups v2 is active:

```bash
stat -fc %T /sys/fs/cgroup
# Expected output: cgroup2fs
```

If it prints `tmpfs`, you are on legacy cgroups v1 and should remove `systemd.unified_cgroup_hierarchy=0` from `/etc/default/grub` and reboot.

## Firewall traps: `ufw`, `firewalld`, and `nftables`

One of the most common Linux lab mysteries is **containers coming up cleanly while traffic between nodes or to the management bridge silently drops**. The culprit is almost always the host's `FORWARD` chain or `br_netfilter`.

### Trap 1: `ufw` default forward policy (`DROP`)

On Ubuntu, `ufw` sets `DEFAULT_FORWARD_POLICY="DROP"` in `/etc/default/ufw`. Docker installs its own `DOCKER-USER` chain, but custom Linux bridges created for Containerlab (`kind: bridge`) or `br_netfilter` traffic can still hit the default `DROP` policy.

Check your forward policy and `DOCKER-USER` chain:

```bash
sudo iptables -L FORWARD -nv --line-numbers
sudo iptables -L DOCKER-USER -nv --line-numbers
```

If you run `ufw` on a dedicated lab VM and need bridge forwarding for Containerlab bridges (`clab-*` or `br-*`), allow traffic on those specific bridge interfaces rather than disabling the firewall globally:

```bash
sudo ufw route allow in on br-+ out on br-+
sudo ufw route allow in on clab-+ out on clab-+
```

### Trap 2: `br_netfilter` intercepting L2 bridge labs

When `br_netfilter` is loaded and `net.bridge.bridge-nf-call-iptables = 1`, pure Layer 2 frames crossing a Linux host bridge are pushed through `iptables`/`nftables` `FORWARD` rules — breaking VLAN/L2 labs unexpectedly. Unless you specifically need L2 firewalling on host bridges (for example Kubernetes), disable bridge-to-iptables interception on a dedicated lab host:

```bash
sudo tee /etc/sysctl.d/99-bridge-l2.conf > /dev/null <<'EOF'
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-arptables = 0
EOF
sudo sysctl --system
```

### Trap 3: `firewalld` on Fedora/RHEL

On Fedora/RHEL, ensure the Docker management bridge (`docker0` and `clab` bridges) is in a trusted zone or reload `firewalld` after installing Docker so Docker's `docker` zone is active:

```bash
sudo firewall-cmd --get-active-zones
```

> [!WARNING]
> Never blindly run `iptables -F` on a cloud VM — flushing rules without setting the default `INPUT` policy to `ACCEPT` can lock you out of SSH. Always test firewall changes cautiously and keep a console/serial recovery path.

## Host health check script

Run this quick audit before deploying any multi-node topology:

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "=== Kernel & Cgroup ==="
uname -r
stat -fc %T /sys/fs/cgroup

echo "=== Docker & Containerlab ==="
docker --version
containerlab version | head -n 5

echo "=== Required Modules ==="
for m in br_netfilter 8021q vxlan wireguard; do
  lsmod | grep -q "^${m}\b" && echo "OK: $m" || echo "MISSING: $m"
done

echo "=== Key Sysctls ==="
sysctl fs.inotify.max_user_instances net.ipv4.ip_forward net.ipv6.conf.all.forwarding
```

## First smoke test

Deploy a minimal two-node Alpine topology, verify packet capture on a `veth` interface, and tear it down cleanly:

```bash
mkdir -p ~/labs/smoke && cd ~/labs/smoke
cat <<'EOF' > twoping.clab.yml
name: twoping
topology:
  nodes:
    h1:
      kind: linux
      image: alpine:3.20
      exec:
        - ip addr add 10.0.0.1/30 dev eth1
        - ip link set eth1 up
    h2:
      kind: linux
      image: alpine:3.20
      exec:
        - ip addr add 10.0.0.2/30 dev eth1
        - ip link set eth1 up
  links:
    - endpoints: ["h1:eth1", "h2:eth1"]
EOF

sudo containerlab deploy -t twoping.clab.yml
sudo containerlab inspect -t twoping.clab.yml
docker exec clab-twoping-h1 ping -c 3 10.0.0.2
sudo ip netns exec clab-twoping-h1 tcpdump -c 2 -ni eth1 icmp &
docker exec clab-twoping-h2 ping -c 1 10.0.0.1
wait
sudo containerlab destroy -t twoping.clab.yml --cleanup
```

Confirm zero orphan containers or lingering bridges remain:

```bash
docker ps -a --filter "name=clab-twoping"
```

## Next

- macOS host path: [Lab Host — macOS](01-lab-host-macos.md)
- Containerlab topology basics: [Containerlab Fundamentals](03-containerlab-fundamentals.md)
- Linux namespaces, `veth`, and `iproute2` primitives: [Linux Networking for Labs](05-linux-networking-for-labs.md)
