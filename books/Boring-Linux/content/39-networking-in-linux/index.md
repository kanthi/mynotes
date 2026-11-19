---
title: 'Part 39 — Networking in Linux'
---

# Networking in Linux

Welcome to **Part 39: Networking in Linux** — *You know networking. Here's how you do networking with Linux.*

Modern network infrastructure runs on Linux. Cloud gateways, hyperscale leaf-spine switches (SONiC, Cumulus Linux), Kubernetes node data planes, carrier-grade NAT appliances, WireGuard meshes, and bare-metal servers all rely on the Linux kernel networking stack. Yet network engineers trained on vendor CLIs (Cisco IOS/NX-OS, Arista EOS, Juniper Junos) often find Linux networking disorienting: there is no single `running-config`, no unified `show run`, and state is spread across kernel subsystems, Netlink APIs, `/proc/sys/net/`, `iproute2`, `nftables`, and multiple userspace network managers.

This part bridges that divide completely. It treats Linux as a full-featured Network Operating System (NOS) — diving from hardware NIC ring buffers, `sk_buff`, and Netlink all the way up through Layer 2 switching, 802.1Q/QinQ VLANs, LACP bonding, policy routing, VRFs, FRRouting (OSPF/BGP/BFD), DHCP, DNS, stateful `nftables` firewalling, NAT, WireGuard/IPsec VPNs, VXLAN/eBPF virtual fabrics, and kernel-level packet forensics. Every chapter pairs deep kernel and protocol mechanics with runnable, self-contained **hands-on labs** built with Linux network namespaces (`ip netns`) that require zero extra physical hardware.

```text
┌──────────────────────────────────────────────────────────────────────────┐
│                  NETWORKING IN LINUX — FULL ARCHITECTURE                 │
│                                                                          │
│   FOUNDATIONS, INTERFACES & LAYERS 1–3                                   │
│   ├── Ch 00: Introduction — Linux as a Network Operating System          │
│   ├── Ch 01: Network Interfaces, Drivers, Offloads & Persistence         │
│   ├── Ch 02: IP Addressing, IPv4/IPv6 Lifecycle & Host Models            │
│   ├── Ch 03: Layer 2 Networking — Ethernet, ARP, NDP & Multicast         │
│   └── Ch 04: Layer 3 Forwarding, sk_buff Pipeline & TCP/IP Internals     │
│                                                                          │
│   SWITCHING, SEGMENTATION & AGGREGATION                                  │
│   ├── Ch 05: VLANs — 802.1Q, 802.1ad (QinQ) & VLAN-Aware Trunks          │
│   ├── Ch 06: Linux Bridges, FDB, STP & Software Switching                │
│   └── Ch 07: Bonding, Teaming & 802.3ad (LACP) Link Aggregation          │
│                                                                          │
│   ROUTING & CORE NETWORK SERVICES                                        │
│   ├── Ch 08: Routing, Policy Routing, VRFs & Dynamic Routing (FRR)       │
│   ├── Ch 09: DHCP — Clients, ISC Kea, dnsmasq & Relays                   │
│   └── Ch 10: DNS — systemd-resolved, Unbound, BIND 9 & Split-DNS         │
│                                                                          │
│   SECURITY, TRANSLATION & TUNNELING                                      │
│   ├── Ch 11: Firewalling — Netfilter Hooks, Conntrack & nftables         │
│   ├── Ch 12: NAT — SNAT, DNAT, Hairpin, CGNAT & Connection Tracking      │
│   └── Ch 13: VPNs — WireGuard, strongSwan IPsec/XFRM, OpenVPN & GRE      │
│                                                                          │
│   VIRTUAL FABRICS, DIAGNOSTICS & CAPSTONE                                │
│   ├── Ch 14: Virtual Networking — netns, VXLAN, OVS, CNI, tc & eBPF/XDP  │
│   ├── Ch 15: Troubleshooting, Packet Forensics & Kernel Drop Hunting     │
│   └── Ch 16: Capstone Lab — Complete Enterprise Linux Network Fabric     │
└──────────────────────────────────────────────────────────────────────────┘
```

## Network Engineer's Translation Table

If you already know how packets move across switches, routers, and firewalls, this table maps traditional network-appliance concepts directly to their Linux equivalents:

| Networking Concept | Vendor Appliance Term | Linux Kernel / Userspace Equivalent |
|---|---|---|
| Physical / logical port | `GigabitEthernet0/1`, `et-0/0/1` | `net_device` (`enp1s0`, `eth0`), managed via `ip link` and `ethtool` |
| Layer 3 interface IP | `ip address ...` | `ip addr add` on a `net_device`; persisted via Netplan / `systemd-networkd` / NM |
| ARP / ND table | `show arp`, `show ipv6 neighbors` | `ip neigh show`, `/proc/sys/net/ipv4/neigh/*` |
| Layer 2 switch | Switching fabric / VLAN bridge | Linux `bridge` device (`ip link add br0 type bridge`) + `bridge fdb` |
| Access / Trunk port | `switchport mode access/trunk` | VLAN-aware bridge (`vlan_filtering 1`) + `bridge vlan add` (PVID / untagged / tagged) |
| Subinterface / SVI | `Gi0/1.100`, `interface Vlan100` | `type vlan` link (`enp1s0.100`) or `br0.100` on bridge `self` |
| Link Aggregation (LAG) | Port-Channel, `ae0`, LACP | `type bond mode 802.3ad` (`bonding` kernel driver) or `teamd` |
| Routing Table (RIB/FIB) | `show ip route` | Kernel FIB trie via `ip route show` (tables `local`, `main`, `default`, custom) |
| Policy-Based Routing | Route-map PBR | `ip rule` + custom routing tables + `nftables` `meta mark` |
| VRF | `vrf definition`, `routing-instance` | `ip link add vrf-blue type vrf table 100` (`l3mdev`) |
| Dynamic Routing | OSPF, BGP, BFD | **FRRouting (FRR)** (`zebra`, `bgpd`, `ospfd`, `bfdd`) synced to kernel FIB |
| Stateful Firewall / ACL | Zone-based firewall, `iptables` | **`nftables`** (`nf_tables`) + `nf_conntrack` |
| NAT / PAT | `ip nat inside/outside` | `nftables` `nat` chains (`prerouting` DNAT, `postrouting` SNAT/MASQUERADE) |
| IPsec / WireGuard VPN | Crypto map, VTI, Tunnel interface | `xfrm` interfaces (`strongSwan`), kernel `wireguard`, `gre`/`ipip` |
| VXLAN / Overlay | NVE / VTEP | `ip link add vxlan100 type vxlan ...` + FRR BGP EVPN or Open vSwitch |
| Virtual Router / Lab | VRF / Virtual Systems | Linux Network Namespaces (`ip netns`) + `veth` virtual patch cables |
| Packet Capture / Debug | SPAN / ERSPAN / `debug ip packet` | `tcpdump`, `tshark`, `nft monitor trace`, `dropwatch`, `pwru` (eBPF) |

## Detailed Section Roadmap

| Chapter | Title | Core Topics & Hands-On Lab |
|---|---|---|
| **Intro** | [Introduction: Networking with Linux](00-introduction-networking-with-linux.md) | Linux NOS architecture, Netlink & `iproute2`, kernel packet flow, disposable `netns` lab harness |
| **01** | [Network Interfaces, Drivers, and Link-Layer Control](01-network-interfaces-drivers-and-link-layer-control.md) | NIC drivers, ring buffers, predictable naming, `ethtool`, offloads, RSS/RPS/XPS, MTU, systemd-networkd/Netplan/NM |
| **02** | [IP Addressing, IPv4/IPv6, and Host Models](02-ip-addressing-ipv4-ipv6-and-host-models.md) | IPv4/IPv6 scopes, SLAAC/DAD/privacy, secondary/anycast IPs, weak vs strong host model, ARP flux controls |
| **03** | [Layer 2 Networking: Ethernet, ARP, and NDP](03-layer-2-networking-ethernet-arp-and-ndp.md) | Ethernet headers, ARP & NDP neighbor state machine, gratuitous ARP, proxy ARP/NDP, IGMP/MLD, LLDP |
| **04** | [Layer 3 Packet Forwarding and IP Stack Internals](04-layer-3-packet-forwarding-and-ip-stack-internals.md) | `sk_buff` lifecycle, forwarding fast-path, PMTUD/ICMPv6, `rp_filter` asymmetric pitfalls, TCP buffers & BBR |
| **05** | [VLANs: 802.1Q, QinQ, and VLAN-Aware Switching](05-vlans-8021q-qinq-and-vlan-aware-switching.md) | 802.1Q/802.1ad tags, subinterfaces, VLAN-aware bridges, PVID/native VLANs, PCP QoS, router-on-a-stick lab |
| **06** | [Linux Bridges and Software Switching](06-linux-bridges-and-software-switching.md) | Kernel bridge internals, FDB learning/aging, STP/RSTP, multicast snooping, port isolation, hairpin, SVIs |
| **07** | [Bonding, Teaming, and Link Aggregation](07-bonding-teaming-and-link-aggregation.md) | All 7 bonding modes, 802.3ad LACP deep dive, hash policies, MII/ARP monitoring, failover tuning, LACP lab |
| **08** | [Routing, Policy Routing, VRFs, and Dynamic Routing](08-routing-policy-routing-vrfs-and-dynamic-routing.md) | Kernel FIB trie, ECMP nexthop groups, PBR (`ip rule`), Linux VRFs (`l3mdev`), FRR OSPF/BGP/BFD lab |
| **09** | [DHCP: Clients, Servers, and Relays](09-dhcp-clients-servers-and-relays.md) | DORA & DHCPv6 (IA_NA/IA_PD), Linux DHCP clients, ISC Kea, dnsmasq, DHCP relays, reservations, PXE lab |
| **10** | [DNS: Resolution, Caching, and Authoritative Services](10-dns-resolution-caching-and-authoritative-services.md) | NSS & glibc stub, `systemd-resolved`, Unbound DNSSEC/DoT, BIND 9 authoritative & views, `dig`/`delv` lab |
| **11** | [Firewalling: Netfilter and nftables](11-firewalling-netfilter-and-nftables.md) | Netfilter hook graph, `conntrack` state machine, `nftables` sets/maps/vmaps/flowtables, tracing, zone firewall lab |
| **12** | [NAT, Connection Tracking, and Address Translation](12-nat-connection-tracking-and-address-translation.md) | SNAT, MASQUERADE, DNAT, hairpin NAT, 1:1 stateless NAT, CGNAT, NAT64, conntrack helpers & exhaustion lab |
| **13** | [VPNs: WireGuard, IPsec, and Encrypted Tunnels](13-vpns-wireguard-ipsec-and-encrypted-tunnels.md) | Encapsulation MTU/MSS math, WireGuard Noise/Cryptokey Routing, strongSwan IKEv2 + XFRM, OpenVPN DCO, GRE lab |
| **14** | [Virtual Networking, Overlays, Containers, and tc/eBPF](14-virtual-networking-overlays-containers-and-tc-ebpf.md) | `netns`, `veth`, `macvlan`/`ipvlan`, `TUN`/`TAP`, VXLAN/EVPN, Geneve, OVS, CNI plumbing, `tc` QoS & XDP/eBPF |
| **15** | [Network Troubleshooting, Packet Forensics, and Observability](15-network-troubleshooting-packet-forensics-and-observability.md) | L1–L7 workflow, `ss`/`nstat`/`ethtool -S`, `tcpdump`/`tshark` BPF, `dropwatch`/`pwru`, 8 break-fix incident labs |
| **16** | [Capstone Lab: Enterprise Linux Network Fabric](16-capstone-hands-on-enterprise-linux-network-lab.md) | End-to-end multi-namespace enterprise fabric integrating routing, VLANs, bonding, DHCP, DNS, NAT, VPN & VXLAN |

> [!TIP]
> **How to use the labs safely:** Every hands-on lab in Part 39 is built inside isolated Linux network namespaces (`ip netns`). Your host's default routing table, SSH session, and physical NICs stay untouched. Run the labs inside an Ubuntu 26.04 LTS VM (or any Linux host with kernel 6.1+ and root privileges) and tear them down cleanly with `ip -all netns delete`.
