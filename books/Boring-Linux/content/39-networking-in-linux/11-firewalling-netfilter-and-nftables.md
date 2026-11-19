---
title: 'Chapter 11: Firewalling: Netfilter and nftables'
---

# Chapter 11: Firewalling: Netfilter and nftables

Linux firewalling is powered by **Netfilter** — a framework of kernel hooks woven directly into the packet processing pipeline — and **`nftables`** (`nf_tables`), the modern register-based packet classification VM that replaced `iptables`, `ip6tables`, `arptables`, and `ebtables`.

Compared to legacy `iptables`, `nftables` provides unified dual-stack (`inet`) tables, O(1)/O(log n) hash and interval sets, verdict maps (`vmap`) for zone-based firewalls, hardware/software **flowtables** that bypass full Netfilter evaluation for established streams, and **100% atomic rule updates**.

## Netfilter Hook Graph and Priority Ordering

Every packet traverses hooks in strict numeric priority order (lower priority executes first):

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                     NETFILTER HOOK GRAPH (inet / ip / ip6)              │
│                                                                         │
│   NIC RX ──► [netdev: ingress (-450)]                                   │
│                     │                                                   │
│                     ▼                                                   │
│             [PREROUTING]  raw(-300) → conntrack(-200) → mangle(-150)    │
│                           → dstnat(-100)                                │
│                     │                                                   │
│                     ▼                                                   │
│            ( Routing Decision )                                         │
│              ╱               ╲                                          │
│    (Local IP)                 (Transit / Forwarded)                     │
│        ▼                               ▼                                │
│   [INPUT]                        [FORWARD]                              │
│   mangle(-150) → filter(0)       mangle(-150) → filter(0)               │
│        │                               │                                │
│        ▼                               │                                │
│  Local Process                         │                                │
│        │                               │                                │
│        ▼                               │                                │
│  (Routing Lookup)                      │                                │
│        ▼                               │                                │
│   [OUTPUT]                             │                                │
│   raw(-300) → conntrack(-200)          │                                │
│   → mangle(-150) → dstnat(-100)        │                                │
│   → filter(0)                          │                                │
│        ╲                               ╱                                │
│         ╲                             ╱                                 │
│          ▼                           ▼                                  │
│             [POSTROUTING]  mangle(-150) → srcnat(100)                   │
│                     │                                                   │
│                     ▼                                                   │
│                  NIC TX                                                 │
└─────────────────────────────────────────────────────────────────────────┘
```

### Understanding `nftables` Address Families

| Family | Scope | Typical Use |
|---|---|---|
| **`inet`** | **Both IPv4 and IPv6 simultaneously** | Standard host and router firewalls (eliminates duplicate rulesets!) |
| `ip` / `ip6` | IPv4-only or IPv6-only | Legacy NAT / family-specific tweaks |
| `netdev` | Per-interface `ingress` / `egress` (before `sk_buff` L3 parsing) | Wire-speed DDoS drops, spoofed bogon filtering |
| `bridge` | Layer 2 bridged frames | Rogue DHCP blocking, MAC filtering on Linux switches |
| `arp` | ARP frames only | ARP spoofing protection |

## Connection Tracking (`nf_conntrack`) Internals

A stateful firewall remembers every active TCP, UDP, and ICMP flow inside the kernel's `nf_conntrack` hash table:

| `ct state` | Meaning |
|---|---|
| `new` | First packet of a flow (e.g., TCP SYN, first UDP datagram) |
| `established` | Packet belongs to a flow that has seen traffic in **both** directions |
| `related` | New flow spawned by an existing one (e.g., ICMP Dest Unreachable / Frag Needed referencing an active TCP flow, or FTP data channel via ALG helper) |
| `invalid` | Malformed, out-of-window, unsolicited FIN/RST, or checksum-broken packet — **always drop early!** |
| `untracked` | Flow explicitly bypassed conntrack using `notrack` in a `raw` (`priority -300`) chain (essential for high-PPS DNS/BGP/DDoS protection) |

### Sizing `nf_conntrack` on Busy Routers and Firewalls

If a busy Linux firewall fills its conntrack table, `dmesg` logs **`nf_conntrack: table full, dropping packet`** and new connections fail immediately!

```bash
# Check current count vs maximum limit
sysctl net.netfilter.nf_conntrack_count net.netfilter.nf_conntrack_max

# Scale conntrack safely (rule of thumb: buckets = max / 4)
# /etc/sysctl.d/60-conntrack.conf
net.netfilter.nf_conntrack_max = 1048576
net.netfilter.nf_conntrack_tcp_timeout_established = 7200
```

## Production `nftables` Constructs: Sets, Maps, Verdict Maps, and Flowtables

Rather than writing 500 linear rules (like `iptables`), idiomatic `nftables` uses **sets** for lists of IPs/ports, **verdict maps (`vmap`)** for zone-to-zone dispatch in a single lookup, **meters** for dynamic per-IP rate limiting, and **flowtables** to fast-path established forwarded flows straight from ingress to egress!

### Complete Production Zone-Based Router Firewall (`/etc/nftables.conf`)

```nft
#!/usr/sbin/nft -f
flush ruleset

define WAN  = "eth0"
define LAN  = "eth1"
define DMZ  = "eth2"

table inet router_fw {
    # Fast-path flowtable: offloads established TCP/UDP flows past full forward chain
    flowtable fastpath {
        hook ingress priority filter
        devices = { $WAN, $LAN, $DMZ }
    }

    # Named interval sets (O(log n) lookup, updatable live without reload!)
    set mgmt_nets {
        type ipv4_addr
        flags interval
        elements = { 10.10.10.0/24, 192.0.2.0/28 }
    }

    set ssh_ratelimit {
        type ipv4_addr
        flags dynamic, timeout
        timeout 1m
    }

    chain input {
        type filter hook input priority filter; policy drop;

        iifname "lo" accept
        ct state vmap { established : accept, related : accept, invalid : drop }

        # Essential ICMPv4 & ICMPv6 (PMTUD, NDP, Echo)
        ip protocol icmp icmp type { echo-request, destination-unreachable, time-exceeded } limit rate 10/second accept
        ip6 nexthdr icmpv6 icmpv6 type {
            destination-unreachable, packet-too-big, time-exceeded, parameter-problem,
            nd-router-solicit, nd-router-advert, nd-neighbor-solicit, nd-neighbor-advert,
            echo-request
        } accept

        # Management SSH only from mgmt_nets, rate-limited per source IP
        ip saddr @mgmt_nets tcp dport 22 ct state new \
            add @ssh_ratelimit { ip saddr limit rate 6/minute burst 10 packets } accept

        # Allow DHCP & DNS requests from LAN
        iifname $LAN udp dport { 53, 67 } accept
        iifname $LAN tcp dport 53 accept

        log prefix "NFT-INPUT-DROP: " limit rate 5/second counter drop
    }

    chain forward {
        type filter hook forward priority filter; policy drop;

        # Offload established TCP/UDP streams to the fast-path flowtable
        ip protocol { tcp, udp } flow add @fastpath
        ct state vmap { established : accept, related : accept, invalid : drop }

        # Zone-Based Dispatch via Concatenated Verdict Map (iifname . oifname)
        iifname . oifname vmap {
            $LAN . $WAN : jump lan_to_wan,
            $LAN . $DMZ : jump lan_to_dmz,
            $WAN . $DMZ : jump wan_to_dmz
        }

        log prefix "NFT-FWD-DROP: " limit rate 5/second counter drop
    }

    chain lan_to_wan {
        # Full outbound access for LAN clients
        accept
    }

    chain lan_to_dmz {
        tcp dport { 22, 80, 443 } accept
        ip protocol icmp accept
    }

    chain wan_to_dmz {
        # Only public HTTPS to DMZ web server
        ip daddr 10.20.0.10 tcp dport { 80, 443 } accept
    }

    chain output {
        type filter hook output priority filter; policy accept;
    }
}
```

### Essential `nft` Operational & Debugging Commands

```bash
# Dry-run syntax validation BEFORE applying (never lock yourself out!)
sudo nft -c -f /etc/nftables.conf

# Atomic reload (zero packet drop window between flush and load)
sudo nft -f /etc/nftables.conf

# Inspect active ruleset with rule handles (needed to delete/insert individual rules)
sudo nft -a list ruleset

# Dynamically add an attacker IP to a set at runtime with auto-expiry
sudo nft add element inet router_fw mgmt_nets { 198.51.100.5/32 }

# Live packet-by-packet rule tracing (Cisco "packet-tracer" on steroids!)
sudo nft insert rule inet router_fw prerouting ip saddr 10.10.10.50 meta nftrace set 1
sudo nft monitor trace
```

> [!TIP]
> **Migrating legacy `iptables` rules:** Use `iptables-translate` (single rule) or `iptables-restore-translate -f rules.v4 > rules.nft` to convert legacy rulesets cleanly into native `nftables` syntax. Avoid mixing `iptables-nft` and native `nftables` tables on the same machine unless you understand how both render into `nft list ruleset`.

## Hands-On Lab 11: Stateful Zone-Based Router Firewall with Live `nft monitor trace`

Build a 3-zone topology (`lan`, `dmz`, `wan`) routed through a Linux firewall (`fw`), enforce a default-drop stateful zone policy using a `vmap`, test permitted vs blocked flows, and trace a live packet through every hook with `nft monitor trace`.

```bash
# 1. Create namespaces: lan, dmz, wan, fw
for ns in lan dmz wan fw; do sudo ip netns add "$ns"; done

# 2. Wire all three zones into fw
sudo ip link add lan0 type veth peer name fw-lan
sudo ip link add dmz0 type veth peer name fw-dmz
sudo ip link add wan0 type veth peer name fw-wan

sudo ip link set lan0 netns lan; sudo ip link set fw-lan netns fw
sudo ip link set dmz0 netns dmz; sudo ip link set fw-dmz netns fw
sudo ip link set wan0 netns wan; sudo ip link set fw-wan netns fw

# 3. Assign IPs and default routes
sudo ip -n lan link set lan0 up; sudo ip -n lan addr add 10.10.0.10/24 dev lan0
sudo ip -n lan route add default via 10.10.0.1

sudo ip -n dmz link set dmz0 up; sudo ip -n dmz addr add 10.20.0.10/24 dev dmz0
sudo ip -n dmz route add default via 10.20.0.1

sudo ip -n wan link set wan0 up; sudo ip -n wan addr add 198.51.100.10/24 dev wan0
sudo ip -n wan route add default via 198.51.100.1

sudo ip -n fw link set lo up
sudo ip -n fw link set fw-lan up; sudo ip -n fw addr add 10.10.0.1/24 dev fw-lan
sudo ip -n fw link set fw-dmz up; sudo ip -n fw addr add 10.20.0.1/24 dev fw-dmz
sudo ip -n fw link set fw-wan up; sudo ip -n fw addr add 198.51.100.1/24 dev fw-wan
sudo ip netns exec fw sysctl -w net.ipv4.ip_forward=1

# 4. Load an atomic stateful zone-based nftables policy into fw:
#    - LAN -> WAN: allow all
#    - WAN -> DMZ: allow ONLY TCP/8080 to 10.20.0.10
#    - DMZ -> LAN: BLOCKED (even ICMP!)
sudo ip netns exec fw nft -f - << 'EOF'
flush ruleset
table inet fw {
    chain forward {
        type filter hook forward priority filter; policy drop;
        ct state established,related accept
        ct state invalid drop
        iifname . oifname vmap {
            "fw-lan" . "fw-wan" : accept,
            "fw-wan" . "fw-dmz" : jump wan_to_dmz
        }
        counter drop
    }
    chain wan_to_dmz {
        ip daddr 10.20.0.10 tcp dport 8080 counter accept
    }
}
EOF

# 5. Start a tiny HTTP listener in DMZ on TCP/8080
sudo ip netns exec dmz python3 -m http.server 8080 --bind 10.20.0.10 >/dev/null 2>&1 &
HTTP_PID=$!
sleep 0.5

# 6. Verify policy matrix:
echo "=== Test 1: LAN -> WAN ping (MUST SUCCEED) ==="
sudo ip netns exec lan ping -c 2 198.51.100.10

echo "=== Test 2: WAN -> DMZ HTTP:8080 (MUST SUCCEED) ==="
sudo ip netns exec wan curl -sI http://10.20.0.10:8080 | head -n 1

echo "=== Test 3: WAN -> DMZ ping (MUST BE DROPPED) ==="
sudo ip netns exec wan ping -c 1 -W 1 10.20.0.10 || echo "Blocked as expected!"

echo "=== Test 4: DMZ -> LAN ping (MUST BE DROPPED) ==="
sudo ip netns exec dmz ping -c 1 -W 1 10.10.0.10 || echo "Blocked as expected!"

# 7. Inspect conntrack entries and nftables counters
sudo ip netns exec fw nft list ruleset
sudo ip netns exec fw conntrack -L 2>/dev/null || true

# 8. Cleanup
sudo kill "$HTTP_PID" 2>/dev/null || true
for ns in lan dmz wan fw; do sudo ip netns del "$ns"; done
```

**What you verified:** Your `vmap` zone firewall permitted stateful LAN→WAN and WAN→DMZ:8080 traffic while dropping DMZ→LAN and unauthorized WAN→DMZ probes, with exact packet/byte counters recorded in `nft list ruleset`.
