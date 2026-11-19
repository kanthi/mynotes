---
title: 'Chapter 4: Layer 3 Packet Forwarding and IP Stack Internals'
---

# Chapter 4: Layer 3 Packet Forwarding and IP Stack Internals

Turning a Linux server into a Layer 3 router takes one sysctl flag — `net.ipv4.ip_forward = 1` — yet operating a reliable, high-throughput Linux router requires deep fluency in how the kernel handles `sk_buff` packets, forwarding validation, Reverse Path Filtering (`rp_filter`), ICMP Path MTU Discovery (PMTUD), redirects, and TCP/UDP socket buffering.

## Inside `sk_buff` and the IPv4/IPv6 Forwarding Pipeline

Every packet inside the Linux kernel is represented by **`struct sk_buff`** (`skb`), a metadata descriptor pointing to a linear packet buffer (`head`, `data`, `tail`, `end`) plus optional paged scatter-gather fragments (`skb_shared_info`). As a packet climbs up or down the stack, the kernel rarely copies payload bytes — it simply moves pointers (`skb_pull`, `skb_push`) to strip or prepend Ethernet, VLAN, IP, and TCP headers in O(1) time.

When an IPv4 packet enters Layer 3 (`ip_rcv()`):

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                    LINUX KERNEL IPv4 FORWARDING PATH                    │
│                                                                         │
│   ip_rcv()                                                              │
│     ├─ Validate IPv4 header length, version, checksum, total length     │
│     ▼                                                                   │
│   NF_INET_PRE_ROUTING  (raw → conntrack → mangle → DNAT)                │
│     ▼                                                                   │
│   ip_rcv_finish() ──► ip_route_input_noref()                            │
│     ├─ Reverse Path Filter check (rp_filter)                            │
│     ├─ Martian source/destination validation                            │
│     ├─ FIB Lookup (RPDB ip rules → routing table trie)                  │
│     │                                                                   │
│     ├─── Local IP? ──► ip_local_deliver() ──► NF_INET_LOCAL_IN ──► Sock │
│     │                                                                   │
│     └─── Transit IP? (requires ip_forward == 1)                         │
│            ▼                                                            │
│          ip_forward()                                                   │
│            ├─ Check TTL <= 1 → send ICMP Time Exceeded & drop           │
│            ├─ Decrement TTL, recompute IPv4 header checksum             │
│            ├─ Check egress MTU & DF flag → ICMP Frag Needed (Type 3/4)  │
│            ├─ Check if input_iface == output_iface → ICMP Redirect?     │
│            ▼                                                            │
│          NF_INET_FORWARD (filter / firewall policy)                     │
│            ▼                                                            │
│          ip_forward_finish() ──► ip_output()                            │
│            ▼                                                            │
│          NF_INET_POST_ROUTING (mangle → SNAT / MASQUERADE)              │
│            ▼                                                            │
│          ip_finish_output2() ──► Neighbor lookup (ARP) ──► dev_queue_xmit│
└─────────────────────────────────────────────────────────────────────────┘
```

### Enabling IPv4 and IPv6 Forwarding Properly

```ini
# /etc/sysctl.d/40-router-forwarding.conf
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
```

> [!IMPORTANT]
> **Subtle per-interface behavior:** In Linux IPv4, setting `net.ipv4.ip_forward = 1` automatically sets `conf.all.forwarding = 1` and `conf.<iface>.forwarding = 1` on all interfaces. In **IPv6**, forwarding is evaluated per incoming interface (`conf.all.forwarding` overrides all interfaces). Remember from [Chapter 2](02-ip-addressing-ipv4-ipv6-and-host-models.md): enabling IPv6 forwarding disables SLAAC Router Advertisements unless `net.ipv6.conf.<wan>.accept_ra = 2` is explicitly configured.

## Reverse Path Filtering (`rp_filter`) and Asymmetric Routing

Reverse Path Filtering (RFC 3704 uRPF) validates that the **source IP** of an incoming packet is reachable via the routing table before allowing it through:

| Value | Mode | Kernel Check | When to Use |
|---|---|---|---|
| `0` | Disabled | No source route validation | Specialized policy-routed appliances / PBR tunnels |
| `1` | **Strict uRPF** | The **best route** back to Source IP must exit the **exact same interface** the packet arrived on | Single-homed hosts, stub access interfaces |
| `2` | **Loose uRPF** | Source IP must be reachable via **any** interface in the routing table | **Multi-homed routers, ECMP, BGP peers, asymmetric paths, WireGuard/VPN gateways** |

> [!CAUTION]
> **The classic `max(all, iface)` trap:** The effective `rp_filter` value on interface `eth0` is **`max(net.ipv4.conf.all.rp_filter, net.ipv4.conf.eth0.rp_filter)`**!
> If `conf.all.rp_filter = 1` (strict) and you set `conf.eth0.rp_filter = 2` (loose), `max(1, 2)` is `2` — wait: **`max(1, 0)` is `1`!** Setting `eth0.rp_filter = 0` while `all.rp_filter = 1` leaves strict filtering **ENABLED**!
> Always set `all.rp_filter = 0` (or `2`) and configure per-interface values explicitly.

Log dropped spoofed or asymmetrically routed packets so they are never invisible:

```bash
sudo sysctl -w net.ipv4.conf.all.log_martians=1
sudo sysctl -w net.ipv4.conf.default.log_martians=1
# Watch martian drops live in kernel logs:
dmesg -w | grep -i martian
```

## Path MTU Discovery (PMTUD), Fragmentation, and ICMP

When a Linux router forwards a packet larger than the egress interface MTU:
- **IPv4 with DF=0 (Don't Fragment clear):** Linux fragments the packet across multiple IPv4 frames.
- **IPv4 with DF=1 (standard for all modern TCP):** Linux drops the packet and sends **ICMP Type 3, Code 4 ("Fragmentation Needed and DF Set")** containing the egress MTU back to the sender.
- **IPv6:** Routers **never fragment** in flight. Linux drops the packet and sends **ICMPv6 Type 2 ("Packet Too Big")** containing the egress MTU.

> [!WARNING]
> **Never block all ICMP/ICMPv6 in your firewall!** Blocking ICMP Type 3 Code 4 (IPv4) or ICMPv6 Type 2 (IPv6) breaks Path MTU Discovery across VPNs, PPPoE, and cloud VXLAN links. Linux also supports **RFC 4821 TCP MTU Probing** (`net.ipv4.tcp_mtu_probing = 1`) as a defense against broken third-party black-hole firewalls.

Inspect cached Path MTUs learned by the Linux kernel:

```bash
# View dynamically cached PMTU values per destination
ip route show cache
ip route get 198.51.100.25
```

### ICMP Redirects

When a router receives a packet on `eth0` and forwards it back out **`eth0`** to a better next-hop on the same subnet, Linux sends an **ICMP Redirect** telling the sender to bypass it. On modern segmented/firewalled networks and hosts, disable redirects to prevent routing-table injection attacks:

```ini
# /etc/sysctl.d/41-icmp-hardening.conf
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0
```

## Transport Stack & Socket Buffer Tuning (BDP & BBR)

High-bandwidth, high-latency links (e.g., 10 Gbps across 50 ms RTT) require enough TCP window space to fill the **Bandwidth-Delay Product (BDP)**:

$$\text{BDP} = \text{Bandwidth} \times \text{RTT} = 10 \times 10^9\text{ bps} \times 0.050\text{ s} \div 8 = 62.5\text{ MB}$$

Linux auto-tunes TCP send/receive buffers up to the ceilings configured in `tcp_rmem`, `tcp_wmem`, `rmem_max`, and `wmem_max`, and pairs them with modern congestion control (**BBR** + **`fq`** pacing qdisc):

```ini
# /etc/sysctl.d/45-tcp-performance.conf
# Socket buffer maximums (64 MiB)
net.core.rmem_max = 67108864
net.core.wmem_max = 67108864
net.ipv4.tcp_rmem = 4096 131072 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864

# Backlog & queue sizing for bursty links
net.core.netdev_max_backlog = 16384
net.core.somaxconn = 8192
net.ipv4.tcp_max_syn_backlog = 8192

# Enable RFC 4821 PMTUD black-hole mitigation
net.ipv4.tcp_mtu_probing = 1

# Enable Fair Queue pacing + BBR congestion control
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
```

Check available and active congestion control algorithms:

```bash
sysctl net.ipv4.tcp_available_congestion_control
sysctl net.ipv4.tcp_congestion_control
```

## Hands-On Lab 4: Three-Hop Router, PMTUD in Action, and Asymmetric `rp_filter` Failure

Build a classic 3-node routed chain (`h1 <-> r1 <-> h2`) plus a second asymmetric path (`r2`) to watch TTL decrement, ICMP Packet Too Big (PMTUD), and strict `rp_filter` silent packet drops firsthand.

```bash
# 1. Create namespaces: h1, r1, r2, h2
for ns in h1 r1 r2 h2; do sudo ip netns add "$ns"; done

# 2. Wire dual transit paths:
#    h1 ---(10.1.0.0/24)--- r1 ---(10.2.0.0/24)--- h2
#    h1 ---(10.3.0.0/24)--- r2 ---(10.4.0.0/24)--- h2
sudo ip link add h1-r1 type veth peer name r1-h1
sudo ip link add r1-h2 type veth peer name h2-r1
sudo ip link add h1-r2 type veth peer name r2-h1
sudo ip link add r2-h2 type veth peer name h2-r2

sudo ip link set h1-r1 netns h1; sudo ip link set h1-r2 netns h1
sudo ip link set r1-h1 netns r1; sudo ip link set r1-h2 netns r1
sudo ip link set r2-h1 netns r2; sudo ip link set r2-h2 netns r2
sudo ip link set h2-r1 netns h2; sudo ip link set h2-r2 netns h2

# 3. Assign IPs and enable forwarding on r1 and r2
sudo ip -n h1 link set h1-r1 up; sudo ip -n h1 addr add 10.1.0.10/24 dev h1-r1
sudo ip -n h1 link set h1-r2 up; sudo ip -n h1 addr add 10.3.0.10/24 dev h1-r2

sudo ip -n r1 link set r1-h1 up; sudo ip -n r1 addr add 10.1.0.1/24 dev r1-h1
sudo ip -n r1 link set r1-h2 up; sudo ip -n r1 addr add 10.2.0.1/24 dev r1-h2
sudo ip netns exec r1 sysctl -w net.ipv4.ip_forward=1

sudo ip -n r2 link set r2-h1 up; sudo ip -n r2 addr add 10.3.0.1/24 dev r2-h1
sudo ip -n r2 link set r2-h2 up; sudo ip -n r2 addr add 10.4.0.1/24 dev r2-h2
sudo ip netns exec r2 sysctl -w net.ipv4.ip_forward=1

sudo ip -n h2 link set h2-r1 up; sudo ip -n h2 addr add 10.2.0.20/24 dev h2-r1
sudo ip -n h2 link set h2-r2 up; sudo ip -n h2 addr add 10.4.0.20/24 dev h2-r2

# 4. Symmetric routing first (via r1)
sudo ip -n h1 route add 10.2.0.0/24 via 10.1.0.1
sudo ip -n h2 route add 10.1.0.0/24 via 10.2.0.1
sudo ip netns exec h1 ping -c 2 10.2.0.20

# 5. Simulate a narrow 1300-byte MTU link between r1 and h2 — observe ICMP Frag Needed (PMTUD)!
sudo ip -n r1 link set r1-h2 mtu 1300
sudo ip -n h2 link set h2-r1 mtu 1300
sudo ip netns exec h1 ping -c 2 -M do -s 1400 10.2.0.20 || true
# Notice ping explicitly prints: "From 10.1.0.1 icmp_seq=1 Frag needed and DF set (mtu = 1300)"
sudo ip -n h1 route get 10.2.0.20    # Shows cached "mtu 1300" learned via PMTUD!

# 6. Now force ASYMMETRIC routing:
#    Forward: h1 -> r1 -> h2 (10.1.0.10 -> 10.2.0.20)
#    Return:  h2 -> r2 -> h1 (route 10.1.0.0/24 via r2!)
sudo ip -n r2 route add 10.1.0.0/24 via 10.3.0.10
sudo ip -n h2 route replace 10.1.0.0/24 via 10.4.0.1

# 7. Turn on STRICT rp_filter (1) on h2 — asymmetric traffic SILENTLY BREAKS!
sudo ip netns exec h2 sysctl -w net.ipv4.conf.all.rp_filter=1
sudo ip netns exec h2 sysctl -w net.ipv4.conf.h2-r1.rp_filter=1
sudo ip netns exec h1 ping -c 2 -W 1 10.2.0.20 || echo "Blocked by strict rp_filter!"

# 8. Switch h2 to LOOSE rp_filter (2) — asymmetric routing immediately recovers!
sudo ip netns exec h2 sysctl -w net.ipv4.conf.all.rp_filter=2
sudo ip netns exec h2 sysctl -w net.ipv4.conf.h2-r1.rp_filter=2
sudo ip netns exec h1 ping -c 2 10.2.0.20

# 9. Cleanup
for ns in h1 r1 r2 h2; do sudo ip netns del "$ns"; done
```

**What you verified:** You observed kernel PMTUD injecting `mtu 1300` into the route cache via ICMP Type 3 Code 4, proved why strict `rp_filter=1` kills asymmetric routing silently, and fixed it with loose `rp_filter=2`.
