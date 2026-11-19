---
title: Lab Host — macOS
---

# Lab Host — macOS

Prepare a **macOS** machine (Apple Silicon or Intel) as a Containerlab control host. Because Containerlab creates Linux network namespaces, `veth` pairs, and bridges directly in the kernel, running network labs on macOS always involves a **Linux virtual machine layer**. Understanding that boundary prevents the most common macOS lab failures: unreachable node IPs, broken packet captures, and silent `amd64` emulation stalls.

## Learning goals

By the end of this chapter you can:

- Choose the right Linux VM and container runtime on macOS (OrbStack, Colima, Lima, or Docker Desktop)
- Decide when to run Containerlab **inside** a Linux VM instead of calling it from macOS directly
- Budget CPU, RAM, and disk space for multi-node FRR, SR Linux, and cEOS topologies
- Navigate Apple Silicon (`arm64`) vs `amd64` container image tradeoffs
- Reach lab nodes cleanly via SSH, port forwarding, and `docker exec`
- Diagnose and fix common macOS host-to-VM networking pitfalls

## Prerequisites

- macOS 13 (Ventura) or newer with Apple Virtualization Framework support
- Admin privileges and [Homebrew](https://brew.sh/) installed
- At least **16 GB RAM** (8 GB works for 2–4 FRR/Alpine nodes; 32 GB+ is comfortable for SR Linux, cEOS, or 10+ node fabrics)
- At least **25–40 GB free SSD space** for NOS container images, VM disks, and packet captures

## Apple Silicon vs Intel tradeoffs

Containerlab itself ships as a native macOS binary, but the **network operating system (NOS) images** you run inside containers are Linux binaries built for a specific CPU architecture.

| NOS / Image | Native `linux/arm64` | `linux/amd64` on Apple Silicon | Practical Verdict on Apple Silicon |
|---|---|---|---|
| `alpine:3.20` / `debian` | Yes | Not needed | Fast, tiny (~10–80 MB RAM) |
| `quay.io/frrouting/frr` | Yes | Not needed | Primary open stack for this book; runs natively at full speed |
| `ghcr.io/nokia/srlinux` | Yes (recent tags) | Rosetta / QEMU fallback | Verify tag manifest; native `arm64` is smooth |
| Arista `cEOS-lab` | Limited / newer releases only | Rosetta 2 (OrbStack/Docker) or QEMU | Prefer native `arm64` cEOS build when available; QEMU emulation is slow and memory-heavy |
| `vrnetlab` VM-based NOS (vEOS, vJunos, CSR) | No (`x86_64` KVM) | TCG software emulation only | **Avoid on Apple Silicon** — no nested KVM for `x86_64` guests; offload to a Linux `x86_64` host or cloud VM |

> [!IMPORTANT]
> Check an image's supported architectures before pulling it into a topology:
> `docker manifest inspect quay.io/frrouting/frr:10.2.1 | grep architecture`
> If `arm64` is missing, your runtime will fall back to Rosetta 2 (fast, Linux user-space only) or QEMU (slow, high CPU).

## Choosing a container runtime and VM path

On macOS, you have two fundamentally different ways to run Containerlab:

1. **Linux VM with Containerlab installed inside the VM (Recommended)** — OrbStack Linux machine, Lima VM, or Multipass/UTM. You SSH into the Linux VM, run `sudo containerlab deploy`, and get **native Linux `veth`, `tcpdump`, `ip netns`, and direct container IP routing** with zero macOS-socket quirks.
2. **Docker-only runtime on macOS** — Docker Desktop or Colima exposing a Docker socket to macOS. Containerlab runs via its container-based wrapper or inside the Colima VM, because native macOS lacks Linux netlink/netns syscalls.

| Runtime | Virtualization | Rosetta 2 (`amd64`) | Direct Lab Subnet Reachability | Licensing & Notes |
|---|---|---|---|---|
| **OrbStack** | Apple Virtualization.framework | Yes (automatic) | Yes — macOS routes directly to container & Linux VM IPs | Fast boot, low idle RAM; free for personal use, paid for commercial |
| **Colima** | Lima + VZ or QEMU | Yes (with `--vm-type=vz --vz-rosetta`) | Via VM SSH or `--network-address` | Fully open source; great CLI-only workflow |
| **Lima** | Apple VZ or QEMU | Yes (`rosetta: enabled`) | Via port forward or `socket_vmnet` | Full control over cloud-init, disk, and kernel modules |
| **Docker Desktop** | Apple VZ | Yes (VirtioFS + Rosetta) | Port publishing only (`-p`) | Familiar GUI; commercial license required for larger orgs |

### Recommended path: OrbStack Linux VM

If you use OrbStack, spin up a dedicated Ubuntu Linux machine for labs rather than relying solely on the Docker engine:

```bash
brew install --cask orbstack
orb create ubuntu:24.04 netlab
orb shell -m netlab
```

Inside the `netlab` Linux VM, follow [Lab Host — Linux](02-lab-host-linux.md) to install Docker Engine and Containerlab natively. OrbStack automatically mounts your macOS home directory at `/mnt/mac/Users/<you>` (or shares the working directory) and lets macOS reach the VM by hostname (`netlab.orb.local`).

### Open-source path: Colima or Lima

With **Colima**, allocate enough CPU and memory up front and enable Apple Virtualization Framework (`vz`) plus Rosetta for `amd64` images:

```bash
brew install colima docker docker-compose
colima start \
  --cpu 4 \
  --memory 8 \
  --disk 60 \
  --vm-type vz \
  --vz-rosetta \
  --network-address
```

Verify Docker is reachable from macOS:

```bash
docker version
docker info --format '{{.Architecture}} {{.MemTotal}} {{.NCPU}}'
```

To run Containerlab cleanly with Colima, install `containerlab` **inside the Colima VM** via `colima ssh`:

```bash
colima ssh
curl -sL https://containerlab.dev/setup | sudo -E bash -s "all"
containerlab version
```

With **Lima**, create a dedicated VM template (`netlab.yaml`) with `vmType: vz`, `rosetta: { enabled: true, binfmt: true }`, 4+ CPUs, 8+ GB RAM, and `containerd`/`docker` provisioned via `provision:` scripts, then run `limactl shell netlab`.

> [!TIP]
> Running Containerlab **inside** an OrbStack, Colima, or Lima Linux VM is almost always cleaner than running a macOS wrapper binary. Your `ip netns`, `bridge`, `tcpdump`, and Wireshark remote capture commands match [Lab Host — Linux](02-lab-host-linux.md) verbatim.

## CPU, RAM, and disk budgeting

Do not give your Linux VM 100% of your Mac's RAM — macOS needs headroom for the host OS, browser, IDE, and Wireshark.

| Host Mac RAM | Recommended VM RAM | Recommended VM vCPUs | Comfortable Lab Size |
|---|---|---|---|
| 16 GB | 6–8 GB | 4 | 6–10 FRR/Alpine nodes, or 2–3 SR Linux nodes |
| 24–32 GB | 12–16 GB | 6 | 12–20 FRR nodes, or 4–6 SR Linux / cEOS nodes |
| 48–64 GB+ | 24–32 GB | 8 | Full 2×4 Clos + border + telemetry stack |

Typical per-node memory footprints (idle to light BGP/OSPF):

- **Alpine Linux host:** 15–40 MB
- **FRR (`quay.io/frrouting/frr`):** 80–150 MB
- **Nokia SR Linux:** 1.0–1.5 GB
- **Arista cEOS-lab:** 1.2–2.0 GB (higher under `amd64` emulation)

Check live consumption inside your VM during a lab:

```bash
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

## Reaching lab nodes: SSH, `docker exec`, and port forwarding

Because Containerlab attaches nodes to a Docker management bridge (`172.20.20.0/24` by default) **inside the Linux VM**, macOS cannot directly `ssh 172.20.20.2` unless your runtime bridges or routes that subnet (OrbStack does; Docker Desktop and default Colima do not).

Three reliable ways to connect from macOS:

### 1. Jump through the Linux VM (SSH `ProxyJump`)

Add an entry to `~/.ssh/config` on macOS so you can SSH directly into lab nodes by name or management IP:

```text
Host netlab
  HostName netlab.orb.local
  User ubuntu

Host clab-* 172.20.20.*
  ProxyJump netlab
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
```

### 2. Local port forwarding for web UIs and gNMI/Grafana

Forward a telemetry or management port from the Linux VM to macOS `localhost`:

```bash
ssh -L 3000:172.20.20.10:3000 -N netlab
```

Or publish ports directly in `topology.clab.yml` under a node's `ports:` list (for example `"8080:80"`), which binds on the Linux VM's interface.

### 3. Direct shell via `docker exec`

For FRR and Alpine nodes, `docker exec` needs no SSH daemon or routing at all:

```bash
docker exec -it clab-twoping-h1 sh
docker exec -it clab-triangle-r1 vtysh
```

### Live packet capture into macOS Wireshark

Pipe `tcpdump` from inside the Linux VM (or a node's network namespace) directly into Wireshark on macOS:

```bash
ssh netlab "sudo ip netns exec clab-triangle-r1 tcpdump -U -i eth1 -w -" \
  | /Applications/Wireshark.app/Contents/MacOS/Wireshark -k -i -
```

## Common macOS pitfalls

| Symptom | Root Cause | Fix |
|---|---|---|
| `containerlab deploy` fails with `operation not supported` on macOS | Running native macOS binary without a Linux kernel for `netlink`/`netns` | Run Containerlab inside OrbStack, Colima, or Lima Linux VM |
| Container starts, immediately exits with `exec format error` | Pulled an `amd64` image on Apple Silicon without Rosetta/QEMU binfmt | Enable Rosetta in OrbStack/Colima/Lima, or use a native `arm64` image tag |
| Bind-mounted `frr.conf` shows permission errors or stale edits | VirtioFS / macOS host mount UID mismatch or file-sync lag | Keep active lab repos inside the Linux VM filesystem (`~/labs/`) and edit via VS Code Remote-SSH |
| `ping 172.20.20.2` times out from macOS Terminal | Docker bridge lives inside the Linux VM, not on macOS | Run tests inside the VM, use `ProxyJump`, or use OrbStack |
| `inotify` / `too many open files` with 8+ nodes | Low `fs.inotify.max_user_instances` inside the Linux VM | Tune `sysctl` inside the VM (see [Lab Host — Linux](02-lab-host-linux.md)) |
| Lab breaks after Mac wakes from sleep | VM clock drift causes BGP hold timers or TLS/RPKI checks to fail | Sync VM clock (`sudo timedatectl set-ntp true` or restart VM) |

## First smoke test

Run this inside your Linux VM to verify end-to-end health before starting the book's labs:

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
sudo containerlab destroy -t twoping.clab.yml --cleanup
```

Expected result: `3 packets transmitted, 3 packets received, 0% packet loss`, followed by zero leftover `clab-twoping-*` containers in `docker ps -a`.

## Next

- Linux host configuration and kernel tuning: [Lab Host — Linux](02-lab-host-linux.md)
- First Containerlab workflows and topology anatomy: [Containerlab Fundamentals](03-containerlab-fundamentals.md)
- Choosing free container images by architecture: [Free Image Matrix](04-free-image-matrix.md)
