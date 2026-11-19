---
title: 'Introduction: Networking with Linux'
---

# Introduction: Networking with Linux

If you already understand subnetting, VLAN trunks, spanning tree, ARP, BGP, NAT, and stateful firewalls, you do not need another textbook explaining what a broadcast domain is. You need to know **how the Linux kernel actually implements those concepts**, where its behavior differs sharply from Cisco IOS, Arista EOS, or Juniper Junos, and how to configure, persist, automate, and debug it under pressure.

This chapter establishes the architectural foundation for the rest of Part 39: how Linux separates the data plane from the control plane, how `iproute2` talks to the kernel over Netlink sockets, how a single packet moves through kernel memory, and how to build a zero-hardware multi-router lab on any Linux machine.

## Why Linux Is a First-Class Network Operating System

In a traditional hardware router, a vendor packages a proprietary control plane on top of an ASIC forwarding engine and exposes a single monolithic CLI. Linux takes the opposite architectural stance: **the kernel provides modular, composable networking primitives, and userspace daemons orchestrate them.**

- **Data plane (kernel space):** Packet RX/TX drivers, Ethernet bridging, 802.1Q VLANs, bonding, IPv4/IPv6 forwarding FIB tries, VRFs (`l3mdev`), Netfilter (`nftables` + `conntrack`), XFRM IPsec, WireGuard, VXLAN/Geneve, Traffic Control (`tc`), and programmable eBPF/XDP programs.
- **Control plane (userspace):** `iproute2` (`ip`, `bridge`, `tc`, `ss`), `nft`, `ethtool`, network configuration managers (`systemd-networkd`, Netplan, NetworkManager), routing daemons (FRRouting, BIRD), DHCP/DNS servers (ISC Kea, Unbound, BIND 9), and VPN daemons (`strongSwan`, `wg-quick`).
- **Hardware acceleration (offload):** Through Linux `switchdev`, TC flower offload, XDP native mode, and SmartNIC/DPU drivers, the exact same `ip route`, `bridge vlan`, and `tc` commands program silicon ASICs directly on white-box switches (Cumulus Linux, SONiC, DENT).

> [!IMPORTANT]
> **Critical mental shift:** On a Cisco/Juniper device, commands mutate a persistent configuration tree that compiles down to forwarding state. In Linux, `ip`, `bridge`, `tc`, and `nft` mutate **live kernel runtime state directly** — nothing survives a reboot unless a userspace manager (Netplan, `systemd-networkd`, NetworkManager, `/etc/nftables.conf`, `/etc/sysctl.d/`, FRR) replays that state on boot.

## The Control-Plane Interface: Netlink and `iproute2`

Legacy tools such as `ifconfig`, `route`, `arp`, `brctl`, and `netstat` (`net-tools`) rely on obsolete `ioctl()` calls and `/proc` text parsing. They fail silently on multiple IPs per interface, policy routing tables, VXLAN, VRFs, modern interface flags, and high-rate updates. **Do not use `net-tools` in modern Linux networking.**

Modern Linux uses **Netlink** (`AF_NETLINK`, specifically `NETLINK_ROUTE` / `rtnetlink`), an asynchronous, message-oriented socket protocol between userspace and the kernel.

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                    USERSPACE CONTROL PLANE                              │
│                                                                         │
│   ip / bridge / tc / ss    FRRouting (zebra)    systemd-networkd / NM   │
│            │                      │                       │             │
│            └──────────────────────┼───────────────────────┘             │
│                                   ▼                                     │
│                     AF_NETLINK (rtnetlink sockets)                      │
│              RTM_NEWLINK / RTM_NEWADDR / RTM_NEWROUTE / ...             │
├───────────────────────────────────┼─────────────────────────────────────┤
│                    LINUX KERNEL NETWORKING STACK                        │
│                                   ▼                                     │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐  │
│   │  net_device  │  │ Neighbor FDB │  │  FIB Trie &  │  │ Netfilter  │  │
│   │  & Link Ops  │  │  ARP / NDP   │  │  Policy Rules│  │ & Conntrack│  │
│   └──────────────┘  └──────────────┘  └──────────────┘  └────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

Because Netlink supports multicast groups, you can watch every kernel networking event in real time — just like `debug` on a router, without CPU-melting console spam:

```bash
# Watch all live link, address, neighbor, and route changes via Netlink
ip monitor all

# Watch only route and neighbor table changes with timestamps
ip -ts monitor route neigh

# Machine-readable JSON output (available across ip, bridge, tc, ss)
ip -j -p addr show
```

### The `iproute2` Command Grammar

Every `iproute2` command follows a consistent object-action grammar:

```bash
ip [OPTIONS] OBJECT { COMMAND | help }
```

| Object | Primary Role | Common Inspection Command |
|---|---|---|
| `link` | Layer 1/2 interfaces (physical, VLAN, bridge, bond, veth, vrf, vxlan) | `ip -br -c link show` |
| `address` (`addr`) | Layer 3 IPv4/IPv6 addresses and lifetimes | `ip -br -c addr show` |
| `neighbor` (`neigh`) | ARP (IPv4) and NDP (IPv6) neighbor cache | `ip -c neigh show` |
| `route` | Routing tables (FIB) and multipath nexthops | `ip -c route show table all` |
| `rule` | Policy-Based Routing (RPDB) rule selector | `ip rule show` |
| `nexthop` | Reusable ECMP nexthop objects and groups (kernel 5.3+) | `ip nexthop show` |
| `netns` | Network namespaces (isolated network stacks) | `ip netns list` |
| `vrf` | VRF membership and command execution | `ip vrf show` |
| `xfrm` | IPsec Security Associations (SAs) and Policies (SPs) | `ip xfrm state` |
| `tunnel` / `fou` | GRE, IPIP, SIT, and Foo-over-UDP tunnels | `ip tunnel show` |

Useful global flags to memorize immediately:

- `-c` (`--color`): Colorizes interface state (`UP`/`DOWN`), IPv4 vs IPv6, and MACs.
- `-br` (`--brief`): Compact tabular view (resembles `show ip interface brief`).
- `-d` (`--details`): Exposes virtual device metadata (VLAN ID, VXLAN VNI, bond mode, bridge STP/VLAN filtering, MTU/qdisc).
- `-s` (`--stats`): Displays RX/TX byte, packet, error, and drop counters (use `-s -s` for extended error breakdown).
- `-j` (`--json`) and `-p` (`--pretty`): Emits clean JSON for scripts and `jq`.
- `-n <netns>`: Executes the command inside a target network namespace without spawning a shell.

## End-to-End Packet Lifecycle in the Linux Kernel

Before diving into individual subsystems, keep this high-level map of how an Ethernet frame becomes either a locally delivered payload or a forwarded packet:

```text
  Wire (PHY)
     │
     ▼
  [NIC Hardware] ──► DMA into RX Ring Buffer ──► Hardware IRQ ──► NAPI Poll (softirq)
     │
     ▼
  [XDP Hook] (Driver-level eBPF: XDP_DROP / XDP_TX / XDP_REDIRECT / XDP_PASS)
     │
     ▼
  [sk_buff allocated] ──► GRO (Generic Receive Offload) coalesces segments
     │
     ▼
  [tc ingress] / [Netfilter ingress] (Layer 2/3 early filter & mirror)
     │
     ▼
  [L2 Handler] ──► VLAN untag? ──► Bridge FDB lookup? ──► Bond slave rx_handler?
     │                                    │
     │ (L2 bridged to another port)       ▼ (L3 destined for local MAC / SVI)
     │                          [Netfilter PREROUTING] (raw → conntrack → mangle → DNAT)
     │                                    │
     │                                    ▼
     │                          [Routing Decision (FIB Lookup + RPDB)]
     │                             ╱                         ╲
     │         (Local host IP)    ╱                           ╲ (Remote subnet + ip_forward=1)
     │                           ▼                             ▼
     │                 [Netfilter INPUT]             [Netfilter FORWARD]
     │                           │                             │
     │                           ▼                             │
     │                 [TCP/UDP/ICMP Stack]                    │
     │                           │                             │
     │                           ▼                             │
     │                 [Userspace Socket]                      │
     │                           │                             │
     │                 (Local TX)│                             │
     │                           ▼                             │
     │                 [Routing Lookup]                        │
     │                           │                             │
     │                           ▼                             │
     │                 [Netfilter OUTPUT]                      │
     │                           ╲                             ╱
     │                            ╲                           ╱
     │                             ▼                         ▼
     │                          [Netfilter POSTROUTING] (mangle → SNAT/MASQUERADE)
     │                                    │
     └────────────────────────────────────┼──────────────────────────────┐
                                          ▼                              ▼
                                [tc egress / qdisc] (fq_codel, HTB, CAKE)
                                          │
                                          ▼
                                [GSO/TSO Segmentation] ──► TX Ring Buffer ──► Wire
```

Every subsequent chapter zooms into one specific box in this pipeline — explaining its kernel data structures, its sysctl knobs, how to observe it, and how it fails.

## Five Linux Behaviors That Surprise Network Engineers

1. **Linux uses the Weak Host Model by default (IPv4):** An IP address assigned to `eth0` belongs to the *host*, not strictly to the *interface*. Linux will answer ARP requests for `eth0`'s IP arriving on `eth1` unless `arp_ignore` and `arp_announce` are tuned (covered in [Chapter 2](02-ip-addressing-ipv4-ipv6-and-host-models.md)).
2. **Routing is disabled by default:** A Linux host will drop transit packets unless `net.ipv4.ip_forward = 1` (and `net.ipv6.conf.all.forwarding = 1` for IPv6) is enabled (covered in [Chapter 4](04-layer-3-packet-forwarding-and-ip-stack-internals.md)).
3. **Asymmetric routing breaks under strict `rp_filter`:** Many distributions default `net.ipv4.conf.all.rp_filter` to `1` (strict) or `2` (loose). Strict Reverse Path Filtering silently drops valid asymmetric traffic before `nftables` ever logs it.
4. **Multiple configuration managers can fight each other:** Running `ip addr add` manually while `NetworkManager` or `systemd-networkd` owns the interface can cause the daemon to wipe your settings on link flap or lease renewal. Always verify who owns a link (`networkctl list`, `nmcli dev status`).
5. **Bridge and VLAN filtering interact with Netfilter:** Depending on whether `br_netfilter` is loaded (`net.bridge.bridge-nf-call-iptables`), pure Layer 2 bridged frames can unexpectedly traverse Layer 3 firewall hooks.

## Building Your Disposable Linux Networking Lab Harness

Rather than spinning up five heavy VMs for every chapter, Linux gives you **network namespaces** (`ip netns`). Each namespace has its own completely isolated interfaces, IP addresses, ARP/NDP tables, routing tables, `nftables` ruleset, sockets, and `/proc/sys/net` tunables — while sharing the host kernel with near-zero memory overhead.

### Lab Prerequisites (Ubuntu 26.04 LTS Reference)

Install the complete networking toolkit once on your lab VM:

```bash
sudo apt update && sudo apt install -y \
  iproute2 iputils-ping iputils-arping iputils-tracepath \
  ethtool bridge-utils vlan tcpdump tshark mtr-tiny iperf3 \
  dnsutils ldnsutils ndisc6 lldpd nftables conntrack \
  wireguard-tools strongswan frr kea-dhcp4-server dnsmasq \
  unbound bind9 openvswitch-switch jq
```

> [!NOTE]
> Disable or stop services like `frr`, `kea-dhcp4-server`, `dnsmasq`, `unbound`, and `bind9` on the host's default namespace if you only want to run them selectively inside lab namespaces:
> ```bash
> sudo systemctl disable --now kea-dhcp4-server dnsmasq unbound named frr 2>/dev/null || true
> ```

### Lab 0: Your First Two-Host Virtual Cable

Build two isolated hosts (`h1` and `h2`) connected by a virtual Ethernet patch cable (`veth`), assign IPs, and inspect Netlink traffic:

```bash
# 1. Create two isolated network namespaces
sudo ip netns add h1
sudo ip netns add h2

# 2. Create a virtual Ethernet patch cable (veth pair)
sudo ip link add veth-h1 type veth peer name veth-h2

# 3. Plug each end into its namespace
sudo ip link set veth-h1 netns h1
sudo ip link set veth-h2 netns h2

# 4. Bring up loopback and link interfaces, and assign IPv4 + IPv6 addresses
sudo ip -n h1 link set lo up
sudo ip -n h1 link set veth-h1 up
sudo ip -n h1 addr add 10.0.12.1/24 dev veth-h1
sudo ip -n h1 addr add fd00:12::1/64 dev veth-h1

sudo ip -n h2 link set lo up
sudo ip -n h2 link set veth-h2 up
sudo ip -n h2 addr add 10.0.12.2/24 dev veth-h2
sudo ip -n h2 addr add fd00:12::2/64 dev veth-h2

# 5. Verify Layer 3 reachability over both IPv4 and IPv6
sudo ip netns exec h1 ping -c 3 10.0.12.2
sudo ip netns exec h1 ping -6 -c 3 fd00:12::2

# 6. Inspect the brief interface and neighbor tables inside h1
sudo ip -n h1 -br -c addr show
sudo ip -n h1 -c neigh show

# 7. Clean up cleanly (deleting a namespace automatically destroys its veth peer)
sudo ip netns del h1
sudo ip netns del h2
```

You just built, verified, and tore down a dual-stack Layer 2/Layer 3 network topology in under two seconds. Every chapter ahead expands this exact technique into VLAN trunks, LACP bonds, multi-area OSPF/BGP fabrics, stateful firewalls, NAT gateways, WireGuard/IPsec tunnels, and VXLAN overlays.
