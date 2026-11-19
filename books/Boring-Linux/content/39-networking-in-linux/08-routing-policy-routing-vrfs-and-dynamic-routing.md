---
title: 'Chapter 8: Routing, Policy Routing, VRFs, and Dynamic Routing'
---

# Chapter 8: Routing, Policy Routing, VRFs, and Dynamic Routing

Linux's routing engine is far more than a single `route add default` table. Beneath the hood lies an ultra-fast **Level-Compressed FIB Trie** (`net/ipv4/fib_trie.c`), a programmable **Routing Policy Database (RPDB)** capable of 255+ independent routing tables, native **ECMP Nexthop Groups**, kernel **VRFs (`l3mdev`)**, and full integration with **FRRouting (FRR)** for BGP, OSPF, IS-IS, and BFD at carrier scale.

## The Linux FIB, Route Types, and Administrative Metrics

Inspect the kernel's routing table and exact per-packet forwarding decision:

```bash
# View main IPv4 and IPv6 tables
ip -c route show
ip -6 -c route show

# View ALL tables including the kernel 'local' table (local IPs, broadcasts, loopback)
ip route show table local
ip route show table all

# Ask the kernel's FIB directly: "How would you forward a packet to 8.8.8.8?"
# (Cisco "show ip cef exact-route" / "show route")
ip route get 8.8.8.8
ip route get 8.8.8.8 from 192.0.2.10 iif enp1s0 mark 0x10
```

### Special Linux Route Types

| Route Type | Behavior | Example |
|---|---|---|
| `unicast` (default) | Standard forwarding via gateway or directly attached device | `ip route add 10.0.0.0/8 via 192.0.2.1` |
| `blackhole` | Silently drops matching packets (ideal for BGP RTBH DDoS mitigation & summary discard routes) | `ip route add blackhole 203.0.113.66/32` |
| `unreachable` | Drops and returns ICMP Host Unreachable | `ip route add unreachable 10.99.0.0/16` |
| `prohibit` | Drops and returns ICMP Administratively Prohibited | `ip route add prohibit 172.16.0.0/12` |
| `throw` | Stops searching current policy table and falls through to the next `ip rule` | `ip route add throw 10.0.0.0/8 table isp2` |

### Route Selection Tie-Breakers

1. **Longest Prefix Match (LPM):** `/25` always beats `/24`, regardless of metric or protocol.
2. **Metric (Administrative Distance / Cost):** Lower numeric `metric` wins (`ip route add default via 10.0.0.1 metric 100` beats `metric 200`).
3. **ECMP (Equal-Cost Multi-Path):** Multiple nexthops with the same prefix and metric load-balance flows across paths.

## ECMP and Modern Nexthop Objects

Linux supports both inline multipath routes and modern reusable **Nexthop Groups** (kernel 5.3+, used by FRR for sub-millisecond failover across thousands of prefixes):

```bash
# Inline weighted ECMP route (2:1 ratio across ISP1 and ISP2)
sudo ip route replace default \
  nexthop via 198.51.100.1 dev eth0 weight 2 \
  nexthop via 203.0.113.1 dev eth1 weight 1

# Modern reusable Nexthop Objects (updating nexthop ID 1 updates ALL routes referencing it!)
sudo ip nexthop add id 1 via 198.51.100.1 dev eth0
sudo ip nexthop add id 2 via 203.0.113.1 dev eth1
sudo ip nexthop add id 100 group 1/2
sudo ip route replace 10.0.0.0/8 nhid 100
```

> [!TIP]
> **Critical ECMP hash tuning:** By default, Linux IPv4 ECMP hashes only **Layer 3** (`src IP, dst IP`) — meaning all connections between two servers use one path! Switch to **Layer 4** (`src/dst IP + src/dst port + proto`) for true per-flow ECMP:
> ```bash
> sudo sysctl -w net.ipv4.fib_multipath_hash_policy=1
> sudo sysctl -w net.ipv6.fib_multipath_hash_policy=1
> ```

## Policy-Based Routing (RPDB & `ip rule`)

Traditional routing looks *only* at the **Destination IP**. Policy-Based Routing (PBR) lets Linux choose *which routing table* to consult based on **Source IP**, **ingress interface (`iif`)**, **fwmark** (set by `nftables`), **IP protocol / port**, or **DSCP/TOS**.

```text
┌─────────────────────────────────────────────────────────────────────────┐
│             ROUTING POLICY DATABASE (RPDB) EVALUATION                   │
│                                                                         │
│   Packet ──► Rule 0:     from all lookup local       (Local IPs?)       │
│                 │ (no match)                                            │
│                 ▼                                                       │
│              Rule 1000:  l3mdev                      (Inside a VRF?)    │
│                 │ (no match)                                            │
│                 ▼                                                       │
│              Rule 10000: from 198.51.100.0/24 lookup isp1 (Custom PBR!) │
│                 │ (no match)                                            │
│                 ▼                                                       │
│              Rule 32766: from all lookup main        (Default FIB)      │
│                 │ (no match)                                            │
│                 ▼                                                       │
│              Rule 32767: from all lookup default                        │
└─────────────────────────────────────────────────────────────────────────┘
```

### Multi-WAN Source-Based Policy Routing Example

When a Linux server has two ISPs (`eth0` = `198.51.100.10/24`, `eth1` = `203.0.113.10/24`), replies to traffic arriving on `eth1` must exit `eth1` (not the default gateway on `eth0`):

```bash
# Optional friendly names in /etc/iproute2/rt_tables (or use numeric table IDs 101, 102)
echo "101 isp1" | sudo tee -a /etc/iproute2/rt_tables
echo "102 isp2" | sudo tee -a /etc/iproute2/rt_tables

# Populate each ISP's dedicated routing table
sudo ip route add 198.51.100.0/24 dev eth0 table isp1
sudo ip route add default via 198.51.100.1 dev eth0 table isp1

sudo ip route add 203.0.113.0/24 dev eth1 table isp2
sudo ip route add default via 203.0.113.1 dev eth1 table isp2

# Add policy rules matching source IP (priority 100 and 101)
sudo ip rule add from 198.51.100.10/32 table isp1 priority 100
sudo ip rule add from 203.0.113.10/32 table isp2 priority 101

# Inspect rules
ip rule show
```

> [!WARNING]
> **Remember `rp_filter` on multi-WAN PBR hosts!** Set `net.ipv4.conf.all.rp_filter=2` (loose) so traffic entering `eth1` is not dropped when the `main` table's default route points out `eth0`.

## Linux VRFs (`l3mdev`)

A Linux **VRF** (Virtual Routing and Forwarding) is a Layer 3 isolation device (`type vrf`) bound to a dedicated routing table. Unlike a full network namespace (`ip netns`), VRFs share the same process tree and management plane — identical to Cisco/Juniper VRFs (e.g., isolating an out-of-band `mgmt` VRF from production data-plane VRFs, or separating overlapping tenant `10.0.0.0/8` networks).

```bash
# 1. Create VRF 'vrf-mgmt' backed by routing table 1000
sudo ip link add vrf-mgmt type vrf table 1000
sudo ip link set vrf-mgmt up

# 2. Install an unreachable fallback route inside table 1000 so unmatched VRF
#    traffic NEVER accidentally leaks into the global 'main' routing table!
sudo ip route add unreachable default metric 4278198272 table 1000

# 3. Enslave management interface eth0 to vrf-mgmt
sudo ip link set eth0 master vrf-mgmt
sudo ip addr add 172.31.0.10/24 dev eth0
sudo ip route add default via 172.31.0.1 dev eth0 table 1000

# 4. Inspect VRF membership and VRF-specific routes
ip vrf show
ip route show vrf vrf-mgmt

# 5. Run processes/commands (ping, ssh, curl, sshd) bound to vrf-mgmt!
sudo ip vrf exec vrf-mgmt ping -c 2 172.31.0.1
```

For services listening across VRFs (e.g., `sshd` or BGP inside VRFs), enable:

```bash
sudo sysctl -w net.ipv4.tcp_l3mdev_accept=1
sudo sysctl -w net.ipv4.udp_l3mdev_accept=1
sudo sysctl -w net.ipv4.raw_l3mdev_accept=1
```

## Dynamic Routing on Linux with FRRouting (FRR)

**FRRouting (FRR)** is the de facto enterprise routing suite on Linux (powering Cumulus Linux, SONiC, AWS/Azure appliances, and telco edge routers). Its **`zebra`** daemon acts as the RIB manager and programs best paths into the Linux kernel FIB over Netlink, while protocol daemons (`bgpd`, `ospfd`, `ospf6d`, `isisd`, `bfdd`) speak standard routing protocols — with a Cisco/Arista-style CLI (`vtysh`).

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                       FRROUTING (FRR) ARCHITECTURE                      │
│                                                                         │
│    [ bgpd ]     [ ospfd ]     [ bfdd ]     [ staticd ]                  │
│        │            │             │             │                       │
│        └────────────┴──────┬──────┴─────────────┘                       │
│                            ▼ (ZAPI IPC)                                 │
│                    [ zebra (RIB Manager) ]  ◄──► vtysh CLI              │
│                            │                                            │
│                            ▼ (AF_NETLINK)                               │
│               [ Linux Kernel FIB & VRF Tables ]                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Enabling FRR Daemons (`/etc/frr/daemons`)

```ini
# /etc/frr/daemons
zebra=yes
bgpd=yes
ospfd=yes
bfdd=yes
```

### Production eBGP + BFD + Route-Map Config (`/etc/frr/frr.conf`)

```text
frr defaults datacenter
hostname leaf01
service integrated-vtysh-config
!
ip prefix-list PL-EXPORT seq 10 permit 10.100.1.0/24
!
route-map RM-OUT permit 10
 match ip address prefix-list PL-EXPORT
!
router bgp 65001
 bgp router-id 192.0.2.1
 bgp log-neighbor-changes
 no bgp ebgp-requires-policy
 neighbor 10.255.0.2 remote-as 65002
 neighbor 10.255.0.2 bfd
 !
 address-family ipv4 unicast
  network 10.100.1.0/24
  neighbor 10.255.0.2 route-map RM-OUT out
  maximum-paths 4
 exit-address-family
!
bfd
 profile fast-failover
  detect-multiplier 3
  transmit-interval 100
  receive-interval 100
!
end
```

Inspect live FRR operational state and verify kernel FIB sync:

```bash
sudo vtysh -c "show bgp summary"
sudo vtysh -c "show ip route"
sudo vtysh -c "show bfd peers"
# Notice routes installed by FRR show 'proto bgp' or 'proto ospf' in the kernel!
ip route show proto bgp
```

## Hands-On Lab 8: VRFs, Policy-Based Routing, and Static/Dynamic Multi-Router Topology

Build a 3-router lab (`r1`, `r2`, `r3`) featuring a multi-tenant **VRF router** (`r2`) with overlapping `10.99.0.0/24` customer subnets in `vrf-red` and `vrf-blue` **plus** Policy-Based Routing on `r1`.

```bash
# 1. Create namespaces: red-host, blue-host, vrf-rtr
for ns in red-host blue-host vrf-rtr; do sudo ip netns add "$ns"; done

# 2. Wire BOTH tenants to vrf-rtr
sudo ip link add red0  type veth peer name rtr-red
sudo ip link add blue0 type veth peer name rtr-blue
sudo ip link set red0  netns red-host;  sudo ip link set rtr-red  netns vrf-rtr
sudo ip link set blue0 netns blue-host; sudo ip link set rtr-blue netns vrf-rtr

# 3. Assign IDENTICAL overlapping IP subnet 10.99.0.10/24 to BOTH red-host and blue-host!
sudo ip -n red-host  link set red0 up
sudo ip -n red-host  addr add 10.99.0.10/24 dev red0
sudo ip -n red-host  route add default via 10.99.0.1

sudo ip -n blue-host link set blue0 up
sudo ip -n blue-host addr add 10.99.0.10/24 dev blue0
sudo ip -n blue-host route add default via 10.99.0.1

# 4. Create two isolated VRFs on vrf-rtr (table 100 = vrf-red, table 200 = vrf-blue)
sudo ip -n vrf-rtr link add vrf-red  type vrf table 100
sudo ip -n vrf-rtr link add vrf-blue type vrf table 200
sudo ip -n vrf-rtr link set vrf-red  up
sudo ip -n vrf-rtr link set vrf-blue up

# Prevent route leakage out of VRFs
sudo ip -n vrf-rtr route add unreachable default metric 4278198272 table 100
sudo ip -n vrf-rtr route add unreachable default metric 4278198272 table 200

# Enslave interfaces and assign the EXACT SAME gateway IP 10.99.0.1/24 to BOTH VRFs!
sudo ip -n vrf-rtr link set rtr-red  master vrf-red up
sudo ip -n vrf-rtr addr add 10.99.0.1/24 dev rtr-red

sudo ip -n vrf-rtr link set rtr-blue master vrf-blue up
sudo ip -n vrf-rtr addr add 10.99.0.1/24 dev rtr-blue

# 5. Verify both overlapping VRFs work simultaneously without conflict!
sudo ip netns exec vrf-rtr ip vrf show
sudo ip netns exec vrf-rtr ip route show vrf vrf-red
sudo ip netns exec vrf-rtr ip route show vrf vrf-blue
sudo ip netns exec vrf-rtr ip vrf exec vrf-red  ping -c 2 10.99.0.10
sudo ip netns exec vrf-rtr ip vrf exec vrf-blue ping -c 2 10.99.0.10

# 6. Notice how ip rule automatically has the l3mdev rule at priority 1000!
sudo ip -n vrf-rtr rule show

# 7. Cleanup
for ns in red-host blue-host vrf-rtr; do sudo ip netns del "$ns"; done
```

**What you verified:** Linux VRFs hosted identical overlapping `10.99.0.0/24` subnets on the same kernel simultaneously using separate FIB tables (`100` and `200`) dispatched by the kernel's `l3mdev` RPDB rule.
