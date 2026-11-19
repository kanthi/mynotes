---
title: 'Chapter 3: Layer 2 Networking: Ethernet, ARP, and NDP'
---

# Chapter 3: Layer 2 Networking: Ethernet, ARP, and NDP

Layer 2 is where packets become Ethernet frames and where Layer 3 IP addresses are bound to Layer 2 MAC addresses. While the protocols themselves (Ethernet II, ARP, IPv6 Neighbor Discovery Protocol, IGMP/MLD, LLDP) are RFC standards, Linux implements them through a unified kernel **Neighbor Subsystem** (`net/core/neighbour.c`) with its own state machine, garbage collector, rate limiters, and cache thresholds.

## Ethernet Frame Processing in Linux

An incoming Ethernet II frame carries:
- **6 bytes:** Destination MAC (`ff:ff:ff:ff:ff:ff` broadcast, `01:00:5e:xx:xx:xx` IPv4 multicast, `33:33:xx:xx:xx:xx` IPv6 multicast, or unicast)
- **6 bytes:** Source MAC
- **4 bytes (optional):** 802.1Q VLAN Tag (`0x8100`) or 802.1ad S-Tag (`0x88a8`)
- **2 bytes:** EtherType (`0x0800` IPv4, `0x0806` ARP, `0x86dd` IPv6, `0x88cc` LLDP, `0x8809` LACP)
- **46–1500+ bytes:** Payload + **4-byte FCS** (checked and stripped by NIC silicon)

NIC hardware normally filters frames whose Destination MAC does not match the NIC's unicast MAC, broadcast, or subscribed multicast list (`ip maddr show`). Putting an interface into **promiscuous mode** (`ip link set dev enp1s0 promisc on`) disables that hardware filter — required by bridges, packet sniffers (`tcpdump`), and hypervisors.

## The Linux Neighbor Subsystem (ARP & NDP)

Linux handles IPv4 **ARP** (`arp_tbl`) and IPv6 **NDP** (`nd_tbl`) through the exact same kernel state machine (`struct neighbour`):

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                LINUX NEIGHBOR CACHE STATE MACHINE                       │
│                                                                         │
│   [Packet needs L2 MAC]                                                 │
│            │                                                            │
│            ▼                                                            │
│      ( INCOMPLETE ) ──(no reply after mcast_solicit)──► ( FAILED )      │
│            │                                                            │
│      (ARP Reply / NA)                                                   │
│            ▼                                                            │
│      ( REACHABLE ) ◄───(upper-layer TCP ACK / confirm)───┐              │
│            │                                             │              │
│   (base_reachable_time_ms expires, ~15–45s)              │              │
│            ▼                                             │              │
│        ( STALE ) ──(new TX packet)──► ( DELAY ) ─────────┘              │
│                                           │ (delay_first_probe_time=5s) │
│                                           ▼                             │
│                                       ( PROBE ) ──(unicast solicit)──►  │
│                                           │           (reply → REACHABLE│
│                                           │            none  → FAILED)  │
└─────────────────────────────────────────────────────────────────────────┘
```

### Neighbor States Explained

| State | Meaning | Operational Signal |
|---|---|---|
| `REACHABLE` | Valid MAC recently confirmed (within randomized 0.5×–1.5× `base_reachable_time_ms`) | Healthy active peer |
| `STALE` | Still usable immediately for TX, but kernel will verify on next send | Completely normal for idle neighbors |
| `DELAY` | Packet sent to `STALE` peer; waiting 5 seconds for upper-layer reachability (e.g., TCP ACK) before sending a probe | Transitional |
| `PROBE` | Sending unicast ARP/NS probes (`ucast_solicit`, default 3) | Watching for dead peer |
| `INCOMPLETE` | Broadcast ARP / multicast NS sent; waiting for first reply | Peer down, wrong VLAN, or L2 blocked |
| `FAILED` | Resolution failed; kernel returns `Host unreachable` (`EHOSTUNREACH`) | Confirmed L2 failure |
| `PERMANENT` | Static manually installed entry (`nud permanent`); never ages out | Static security pinning |
| `NOARP` | Point-to-point/tunnel interface that needs no MAC resolution | WireGuard / `tun0` |

### Managing Neighbors with `ip neigh`

```bash
# Show IPv4 and IPv6 neighbor tables with state
ip -c neigh show
ip -4 neigh show dev enp1s0
ip -6 neigh show dev enp1s0

# Show detailed timers (used/confirmed/updated seconds) and reference counts
ip -s neigh show

# Pin a static ARP/NDP entry (prevents ARP spoofing for a critical gateway)
sudo ip neigh replace 192.0.2.1 lladdr 00:11:22:33:44:55 nud permanent dev enp1s0

# Delete a single stale/poisoned entry
sudo ip neigh del 192.0.2.50 dev enp1s0

# Flush all dynamic entries on an interface (leaves PERMANENT entries intact)
sudo ip neigh flush dev enp1s0
```

## Neighbor Table Sizing and Garbage Collection (`neigh/default/gc_thresh*`)

On busy Linux routers, large flat VLANs, or Kubernetes nodes with thousands of peers, the default ARP/NDP cache limits (`gc_thresh1=128`, `gc_thresh2=512`, `gc_thresh3=1024`) are dangerously small. When the table hits `gc_thresh3`, Linux prints **`neighbour: arp_cache: neighbor table overflow!`** in `dmesg` and drops packets!

| Parameter | Default | Role |
|---|---|---|
| `gc_thresh1` | `128` | Minimum entries below which garbage collection does not run |
| `gc_thresh2` | `512` | Soft maximum; GC aggressively cleans after 5 seconds above this |
| `gc_thresh3` | `1024` | Hard maximum; new neighbor allocations fail with `ENOBUFS` |

Tune both IPv4 and IPv6 tables on large Layer 2 segments and Linux routers:

```ini
# /etc/sysctl.d/30-neighbor-scale.conf
net.ipv4.neigh.default.gc_thresh1 = 2048
net.ipv4.neigh.default.gc_thresh2 = 4096
net.ipv4.neigh.default.gc_thresh3 = 8192

net.ipv6.neigh.default.gc_thresh1 = 2048
net.ipv6.neigh.default.gc_thresh2 = 4096
net.ipv6.neigh.default.gc_thresh3 = 8192
```

Check live neighbor cache pressure and lookup stats with:

```bash
cat /proc/net/stat/arp_cache
cat /proc/net/stat/ndisc_cache
```

## Gratuitous ARP, Proxy ARP, and Proxy NDP

### Gratuitous ARP (GARP)

A Gratuitous ARP is an unsolicited ARP Request or Reply where Sender IP == Target IP. HA routers (Keepalived/VRRP) and live-migrated VMs transmit GARPs so upstream switches update their MAC FDB and neighbors update their ARP cache immediately.

- Send GARP manually with `arping`:
  ```bash
  # -U sends unsolicited ARP Request (update neighbors), -A sends ARP Reply
  sudo arping -U -I enp1s0 -c 3 192.0.2.10
  ```
- By default, Linux ignores unsolicited GARP for IPs **not already** in its ARP table (prevents table flooding), and accepts updates only for existing entries unless `arp_accept = 1` is enabled:
  ```bash
  sudo sysctl -w net.ipv4.conf.enp1s0.arp_accept=1
  ```

### Proxy ARP and Proxy NDP

Proxy ARP makes a Linux router answer ARP requests on Interface A on behalf of IP addresses routed out Interface B — useful for legacy flat-subnet migrations, point-to-point `/32` container links (e.g., Calico), or bridging IPv6 `/64` subnets across non-routed cloud links:

```bash
# Enable Proxy ARP on an interface (answers ARP for any IP routed out a DIFFERENT interface)
sudo sysctl -w net.ipv4.conf.enp1s0.proxy_arp=1

# Proxy ARP on the SAME interface (hairpin/PVLAN gateway scenarios)
sudo sysctl -w net.ipv4.conf.enp1s0.proxy_arp_pvlan=1

# Specific IPv4 proxy entry (without enabling blanket proxy_arp)
sudo ip neigh add proxy 192.0.2.99 dev enp1s0

# IPv6 Proxy NDP (requires explicit proxy entries or ndppd daemon)
sudo sysctl -w net.ipv6.conf.enp1s0.proxy_ndp=1
sudo ip -6 neigh add proxy 2001:db8:1::99 dev enp1s0
```

## Layer 2 Multicast (IGMP / MLD) and Link Discovery (LLDP)

### Inspecting L2/L3 Multicast Memberships

```bash
# Link-layer multicast MACs subscribed on each interface
ip maddr show dev enp1s0

# Active IPv4 IGMP group memberships
cat /proc/net/igmp

# Active IPv6 MLD group memberships
cat /proc/net/igmp6
```

### LLDP (IEEE 802.1AB) Neighbor Discovery

Network engineers live on `show lldp neighbors`. On Linux, run **`lldpd`** so your server advertises its hostname, management IP, and port names to top-of-rack switches — and lets you see the exact switch port you are cabled into:

```bash
sudo apt install -y lldpd
sudo systemctl enable --now lldpd

# Equivalent of Cisco "show lldp neighbors detail"
sudo lldpcli show neighbors details
```

> [!TIP]
> Some Intel NICs (like X710 / `i40e`) have an internal firmware LLDP agent that intercepts LLDP frames before `lldpd` ever sees them! If `lldpcli show neighbors` is empty on an Intel server NIC, disable the NIC firmware LLDP agent:
> ```bash
> sudo ethtool --set-priv-flags enp1s0f0 disable-fw-lldp on
> ```

## Hands-On Lab 3: Neighbor State Machine, Proxy ARP, and IPv6 NDP Forensics

Observe ARP & IPv6 NDP state transitions, test static pinning, and implement a Linux Proxy ARP router bridging two segments that share one logical `/24`.

```bash
# 1. Topology: hostA (10.30.0.10/24) <--veth--> router <--veth--> hostB (10.30.0.20/24)
#    Notice hostA and hostB believe they are on the SAME /24, separated by an L3 router!
sudo ip netns add hostA
sudo ip netns add router
sudo ip netns add hostB

sudo ip link add a0 type veth peer name r-a
sudo ip link add b0 type veth peer name r-b

sudo ip link set a0 netns hostA
sudo ip link set r-a netns router
sudo ip link set r-b netns router
sudo ip link set b0 netns hostB

# 2. Configure hostA and hostB with NO default gateway (they will ARP directly for each other)
sudo ip -n hostA link set a0 up
sudo ip -n hostA addr add 10.30.0.10/24 dev a0
sudo ip -n hostA addr add fd00:30::10/64 dev a0

sudo ip -n hostB link set b0 up
sudo ip -n hostB addr add 10.30.0.20/24 dev b0
sudo ip -n hostB addr add fd00:30::20/64 dev b0

# 3. Configure router with forwarding + /32 host routes to each side
sudo ip -n router link set r-a up
sudo ip -n router link set r-b up
sudo ip -n router addr add 10.30.0.1/32 dev r-a
sudo ip -n router addr add 10.30.0.1/32 dev r-b
sudo ip -n router route add 10.30.0.10/32 dev r-a
sudo ip -n router route add 10.30.0.20/32 dev r-b
sudo ip netns exec router sysctl -w net.ipv4.ip_forward=1

# 4. Before Proxy ARP: ping fails and hostA's ARP entry goes INCOMPLETE/FAILED
sudo ip netns exec hostA ping -c 2 -W 1 10.30.0.20 || true
sudo ip -n hostA -c neigh show

# 5. Enable Proxy ARP on both router interfaces
sudo ip netns exec router sysctl -w net.ipv4.conf.r-a.proxy_arp=1
sudo ip netns exec router sysctl -w net.ipv4.conf.r-b.proxy_arp=1

# 6. Now ping succeeds! Inspect hostA's neighbor table: 10.30.0.20 resolves to router's r-a MAC!
sudo ip netns exec hostA ping -c 3 10.30.0.20
sudo ip -n hostA -c neigh show
sudo ip -n router -br link show dev r-a

# 7. Observe IPv6 NDP using ndisc6 / ping6 on a direct segment (router r-a <-> hostA)
sudo ip -n router addr add fd00:30::1/64 dev r-a
sudo ip netns exec hostA ping -6 -c 2 fd00:30::1
sudo ip -n hostA -6 -s neigh show

# 8. Cleanup
sudo ip netns del hostA
sudo ip netns del router
sudo ip netns del hostB
```

**What you verified:** Linux's neighbor cache transitioned cleanly from `FAILED` to `REACHABLE` once Proxy ARP answered across the L3 boundary, mapping `10.30.0.20` to the router's `r-a` MAC address.
