---
title: 'Chapter 1: Network Interfaces, Drivers, and Link-Layer Control'
---

# Chapter 1: Network Interfaces, Drivers, and Link-Layer Control

Every packet entering or leaving a Linux machine crosses a kernel object called `struct net_device`. Whether it is a 100 GbE Mellanox/Intel PCIe adapter, a VirtIO paravirtual NIC inside KVM, a loopback device, or a virtual `veth` pipe, Linux exposes it through a uniform link-layer abstraction.

This chapter covers how Linux discovers and names NICs, how drivers move frames between silicon and RAM using DMA ring buffers and NAPI, how to tune link parameters and hardware offloads with `ethtool`, how MTU/MSS math works, and how to persist interface configurations cleanly with `systemd-networkd`, Netplan, and NetworkManager.

## Anatomy of a Linux Network Interface

Inside the kernel, each interface is represented by `struct net_device` (defined in `include/linux/netdevice.h`), containing:

- **Identifiers:** Interface name (`enp1s0`), numeric `ifindex` (used internally by Netlink, routes, and sockets), MAC address (`dev_addr`), and operational flags (`UP`, `LOWER_UP`, `BROADCAST`, `MULTICAST`, `PROMISC`, `NOARP`).
- **Driver operations (`net_device_ops`):** Function pointers (`ndo_open`, `ndo_stop`, `ndo_start_xmit`, `ndo_change_mtu`, `ndo_set_mac_address`) implemented by the NIC driver (e.g., `ixgbe`, `i40e`, `mlx5_core`, `virtio_net`, `e1000e`).
- **Queues:** One or more RX and TX hardware descriptor rings (`netdev_rx_queue`, `netdev_queue`) plus an egress queuing discipline (`Qdisc`).

### Administrative State vs Operational State

Network engineers recognize the Cisco `up/down` vs `up/up` distinction. Linux exposes the same split in `ip link show`:

```text
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 52:54:00:1a:2b:3c brd ff:ff:ff:ff:ff:ff
```

| Flag / Field | Meaning | Cisco IOS Equivalent |
|---|---|---|
| `UP` (inside `<...>`) | Administrative state is enabled (`ip link set dev enp1s0 up`) | `no shutdown` |
| `LOWER_UP` (inside `<...>`) | Physical carrier / Layer 1 link detected | Line protocol `up` |
| `NO-CARRIER` | Admin UP, but no physical signal / peer link down | `up / down` |
| `state UP` | RFC 2863 `operstate` (`UP`, `DOWN`, `UNKNOWN`, `LOWERLAYERDOWN`, `DORMANT`) | Operational status |

> [!NOTE]
> Virtual interfaces like `lo` and `dummy0` show `state UNKNOWN` when administratively up because they have no physical carrier sensor. That is completely normal and does not indicate a fault.

## Predictable Network Interface Naming

Historically, Linux named Ethernet interfaces `eth0`, `eth1`, `eth2` in whichever order kernel drivers finished PCI probing. On multi-NIC servers, adding a card or changing kernel versions could swap `eth0` and `eth1` across reboots — breaking firewalls and bonding silently.

`systemd-udevd` solves this with **Predictable Network Interface Names** (configured via `/usr/lib/systemd/network/99-default.link` and customizable via `/etc/systemd/network/10-*.link`):

| Prefix / Pattern | Meaning | Example |
|---|---|---|
| `en` / `wl` / `ww` | Ethernet / Wireless LAN / Wireless WWAN | `enp3s0`, `wlp2s0` |
| `eno<index>` | Firmware/BIOS onboard index | `eno1` |
| `ens<slot>` | PCI Express hotplug slot number | `ens192`, `ens3` |
| `enp<bus>s<slot>[f<func>]` | Physical PCI bus/slot/function topology | `enp1s0f0`, `enp1s0f1` |
| `enx<MAC>` | MAC-address-based naming (common for USB NICs) | `enx00e04c6801a2` |

Inspect the exact naming properties discovered by `udev` for any interface:

```bash
udevadm info -q property -p /sys/class/net/enp1s0 | grep -E 'ID_NET_NAME|ID_PATH|ID_NET_DRIVER'
```

To pin a custom, role-based interface name (e.g., `wan0` or `uplink-a`) deterministically by MAC address or PCI path, create `/etc/systemd/network/10-wan0.link`:

```ini
[Match]
MACAddress=52:54:00:1a:2b:3c

[Link]
Name=wan0
Description=Primary ISP Uplink
MTUBytes=1500
```

Regenerate initramfs (`sudo update-initramfs -u`) if the interface is needed in early boot, and reboot for `.link` renames to apply cleanly.

## Hardware Ring Buffers, Interrupts, and NAPI

At 10 Gbps (up to 14.88 million 64-byte packets per second), firing one CPU hardware interrupt per packet would saturate every core in interrupt overhead (receive livelock). Linux solves this with **DMA Ring Buffers** and **NAPI (New API)**:

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                    NIC RX PATH: DMA RING + NAPI                         │
│                                                                         │
│   Wire ──► NIC PHY/MAC ──► DMA write into RX Ring Buffer (RAM)          │
│                                   │                                     │
│                                   ▼ (1st packet arrives)                │
│                            Hardware IRQ (MSI-X)                         │
│                                   │                                     │
│                      Top Half: disable NIC RX IRQ,                      │
│                      schedule NET_RX_SOFTIRQ                            │
│                                   │                                     │
│                                   ▼                                     │
│                      NAPI Poll Loop (ksoftirqd / softirq)               │
│                      Drains up to netdev_budget (300) packets           │
│                      or 2 ms budget per cycle, then re-enables IRQ      │
└─────────────────────────────────────────────────────────────────────────┘
```

### Inspecting and Tuning NICs with `ethtool`

`ethtool` is the primary tool for Layer 1/2 hardware inspection and tuning:

```bash
# Link speed, duplex, auto-negotiation, transceiver/port type
sudo ethtool enp1s0

# Driver name, version, firmware version, and PCI bus address
sudo ethtool -i enp1s0

# Hardware RX/TX ring buffer maximums vs current allocations
sudo ethtool -g enp1s0
sudo ethtool -G enp1s0 rx 4096 tx 4096

# Multi-queue channels (combined / rx / tx queues mapped to MSI-X IRQs)
sudo ethtool -l enp1s0
sudo ethtool -L enp1s0 combined 8

# Interrupt coalescing (trade microseconds of latency for lower CPU interrupts)
sudo ethtool -c enp1s0
sudo ethtool -C enp1s0 adaptive-rx on adaptive-tx on

# Hardware NIC statistics (CRC errors, rx_missed_errors, ring drops, pause frames)
sudo ethtool -S enp1s0 | grep -iE 'err|drop|miss|crc|fifo|over|pause'

# Physically blink the NIC LED for 15 seconds to identify a port in a rack
sudo ethtool -p enp1s0 15
```

> [!WARNING]
> **Ring buffer sizing trade-off:** Increasing RX ring sizes (`ethtool -G`) absorbs microbursts and prevents `rx_missed_errors` / `rx_no_buffer_count`, but oversized TX rings increase bufferbloat and tail latency. Size RX rings generously on bursty receivers; keep TX rings moderate and rely on BQL (Byte Queue Limits) and `fq_codel`.

## Hardware Offloads and Multi-Core Packet Steering

Modern NICs and the Linux kernel work together to avoid per-packet CPU bottlenecks:

| Mechanism | Direction | What It Does | Caution / Router Caveat |
|---|---|---|---|
| **Checksum offload (`rx`/`tx`)** | RX & TX | NIC computes IP/TCP/UDP checksums in silicon | `tcpdump` on the local host shows "bad checksum" on outgoing packets because NIC fills it *after* capture |
| **TSO** (TCP Segmentation Offload) | TX | Kernel hands a 64 KiB TCP super-frame to NIC; NIC slices into MTU segments | Disable if NIC firmware has TSO bugs; fine on hosts |
| **GSO** (Generic Segmentation Offload) | TX | Software fallback to TSO deferred right before NIC driver | Keeps routing/netfilter work amortized |
| **GRO** (Generic Receive Offload) | RX | Reassembles incoming TCP/UDP segments in software before upper stack | **Safe for routers** (preserves headers cleanly for re-segmentation) |
| **LRO** (Large Receive Offload) | RX | Hardware coalescing of incoming TCP packets | **Disable on Linux routers/bridges** (`ethtool -K ... lro off`) — breaks forwarding header fidelity |
| **RSS** (Receive Side Scaling) | RX | Hardware hashes flow tuple (src/dst IP + port) across multiple RX queues/CPUs | Verify IRQ affinity via `/proc/interrupts` |
| **RPS / RFS** | RX | Software flow steering for single-queue NICs | Configured in `/sys/class/net/<dev>/queues/rx-*/rps_cpus` |
| **XPS** | TX | Maps CPUs to dedicated TX queues to avoid lock contention | Configured in `/sys/class/net/<dev>/queues/tx-*/xps_cpus` |

Inspect and toggle offload features:

```bash
# View all offload states ([fixed] means hardware/driver cannot toggle it)
sudo ethtool -k enp1s0

# Disable LRO on a Linux router/firewall interface
sudo ethtool -K enp1s0 lro off
```

## Interface Management with `ip link`

Common runtime link operations:

```bash
# Bring an interface up or down
sudo ip link set dev enp1s0 up
sudo ip link set dev enp1s0 down

# Change MTU (e.g., jumbo frames on a datacenter storage/underlay fabric)
sudo ip link set dev enp1s0 mtu 9000

# Change MAC address (interface must usually be DOWN first on physical NICs)
sudo ip link set dev enp1s0 down
sudo ip link set dev enp1s0 address 02:aa:bb:cc:dd:ee
sudo ip link set dev enp1s0 up

# Add a human-readable interface description (alias) visible in ip link & SNMP
sudo ip link set dev enp1s0 alias "Uplink-Spine01-Eth1/12"

# Create a software dummy interface (Linux equivalent of Cisco Loopback0/1/2)
sudo ip link add loopback0 type dummy
sudo ip link set loopback0 up
sudo ip addr add 192.0.2.1/32 dev loopback0

# Inspect detailed error and drop statistics
ip -s -s link show dev enp1s0
```

### Reading `ip -s -s link` Counters Accurately

| Counter | Meaning | Typical Root Cause |
|---|---|---|
| `RX errors / crc` | Physical frame corruption | Bad SFP optic, damaged cable, duplex mismatch |
| `RX missed` | NIC FIFO overflow before DMA | CPU softirq starvation or RX ring too small |
| `RX dropped` | Kernel dropped frame after DMA | Unknown protocol, VLAN mismatch, `softnet_stat` backlog overflow, no socket buffer |
| `TX carrier` | Carrier lost during transmission | Cable unplugged, switch port flapping |
| `TX dropped` | Egress queue (`qdisc`) drop | Output link congested or `txqueuelen` / qdisc limit exceeded |

## MTU, MRU, and MSS Mechanics

MTU (Maximum Transmission Unit) specifies the largest Layer 3 IP payload (excluding the 14-byte Ethernet header and 4-byte FCS) an interface can transmit in one frame:

- Standard Ethernet: **1500 bytes** → IPv4 TCP MSS = `1500 - 20 (IPv4) - 20 (TCP) = 1460 bytes`; IPv6 TCP MSS = `1500 - 40 (IPv6) - 20 (TCP) = 1440 bytes`.
- PPPoE: **1492 bytes** (8-byte PPPoE overhead).
- WireGuard over IPv4: **1420 bytes** (60-byte IPv4/UDP/WG overhead; 80 bytes over IPv6).
- VXLAN over IPv4: **1450 bytes** inside the overlay unless underlay MTU is raised to **≥ 1550** (or 9000 jumbo frames).

> [!CAUTION]
> **Symmetric MTU rule:** Every device sharing a Layer 2 broadcast domain MUST agree on MTU. If Host A uses MTU 9000 and Switch/Host B uses MTU 1500, small pings (`64 B`) succeed while bulk SSH/TLS/HTTP transfers hang silently — the classic MTU black hole.

## Persisting Configuration Across Reboots

Choose **one** network manager per interface — never let two fight over the same NIC:

| Manager | Best Fit | Config Location |
|---|---|---|
| **Netplan** | Ubuntu Server/Desktop (declarative frontend generating `systemd-networkd` or NM config) | `/etc/netplan/*.yaml` |
| **systemd-networkd** | Servers, routers, containers, headless appliances (lightweight, fast, supports VLAN/bridge/bond/VRF/VXLAN/WireGuard natively) | `/etc/systemd/network/*.{link,netdev,network}` |
| **NetworkManager** | RHEL/Fedora/Workstations, Wi-Fi, dynamic VPN profiles, CLI (`nmcli`) | `/etc/NetworkManager/system-connections/*.nmconnection` |
| **ifupdown2** | White-box switches (Cumulus Linux, Debian routers) | `/etc/network/interfaces` |

### Persistence Example 1: Netplan (Ubuntu 26.04 LTS)

```yaml
# /etc/netplan/10-uplink.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0:
      mtu: 9000
      addresses:
        - 198.51.100.10/24
        - 2001:db8:100::10/64
      routes:
        - to: default
          via: 198.51.100.1
      nameservers:
        addresses: [1.1.1.1, 9.9.9.9]
```

Always test Netplan safely on remote hosts with automatic rollback:

```bash
sudo chmod 600 /etc/netplan/10-uplink.yaml
sudo netplan try --timeout 60    # Reverts automatically if you lose SSH!
sudo netplan apply               # Commit permanently once verified
```

### Persistence Example 2: Native `systemd-networkd`

```ini
# /etc/systemd/network/20-enp1s0.network
[Match]
Name=enp1s0

[Link]
MTUBytes=9000

[Network]
Address=198.51.100.10/24
Address=2001:db8:100::10/64
Gateway=198.51.100.1
DNS=1.1.1.1
```

Apply and verify without rebooting:

```bash
sudo networkctl reload
networkctl status enp1s0
```

## Hands-On Lab 1: Link States, MTU Black Holes, and `dummy` Loopbacks

In this lab, you will simulate a mismatched MTU link, observe fragmentation vs DF-bit black-holing, and create router loopback (`dummy`) interfaces.

```bash
# 1. Create two namespaces connected via veth
sudo ip netns add r1
sudo ip netns add r2
sudo ip link add eth0-r1 type veth peer name eth0-r2
sudo ip link set eth0-r1 netns r1
sudo ip link set eth0-r2 netns r2

# 2. Configure interfaces and add a dummy loopback on r1
sudo ip -n r1 link set lo up
sudo ip -n r1 link set eth0-r1 up
sudo ip -n r1 addr add 10.1.1.1/30 dev eth0-r1
sudo ip -n r1 link add lo0 type dummy
sudo ip -n r1 link set lo0 up
sudo ip -n r1 addr add 192.0.2.1/32 dev lo0

sudo ip -n r2 link set lo up
sudo ip -n r2 link set eth0-r2 up
sudo ip -n r2 addr add 10.1.1.2/30 dev eth0-r2

# 3. Deliberately create an MTU mismatch: r1=1500, r2=1400
sudo ip -n r2 link set eth0-r2 mtu 1400

# 4. Small ping succeeds
sudo ip netns exec r1 ping -c 2 -M do -s 1300 10.1.1.2

# 5. Large DF ping (1472 + 28 IP/ICMP = 1500 bytes) fails silently at r2's RX MTU!
sudo ip netns exec r1 ping -c 2 -W 1 -M do -s 1472 10.1.1.2 || echo "Dropped due to MTU mismatch!"

# 6. Inspect RX error/drop counters on r2 and verify with ip -s -s link
sudo ip -n r2 -s -s link show dev eth0-r2

# 7. Fix MTU symmetry and re-verify
sudo ip -n r1 link set eth0-r1 mtu 1400
sudo ip netns exec r1 ping -c 2 -M do -s 1372 10.1.1.2

# 8. Cleanup
sudo ip netns del r1
sudo ip netns del r2
```

**What you verified:** Even though Layer 1 (`LOWER_UP`) and small pings worked, an asymmetric MTU silently dropped full-sized DF packets until both sides matched. Always check `ip -br link` MTUs across both ends of a link.
