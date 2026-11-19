---
title: 'Chapter 12: NAT, Connection Tracking, and Address Translation'
---

# Chapter 12: NAT, Connection Tracking, and Address Translation

Network Address Translation (NAT) in Linux is built directly on top of **`nf_conntrack`**. That architectural detail explains almost every NAT subtlety in Linux: **only the very first packet (`ct state new`) of a connection ever hits your `nftables` NAT chains!** Every subsequent packet in that flow — and every return packet in the reverse direction — is translated automatically by the kernel's connection tracking table.

## Where NAT Happens in the Netfilter Pipeline

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                    LINUX NAT HOOK EXECUTION ORDER                       │
│                                                                         │
│   Inbound Packet                                                        │
│         │                                                               │
│         ▼                                                               │
│   [PREROUTING: type nat, priority dstnat (-100)]                        │
│     • DNAT (Port Forwarding / Load Balancing / Redirect)                │
│     • Runs BEFORE Routing Decision so kernel routes to NEW dst IP!      │
│         │                                                               │
│         ▼                                                               │
│   ( Routing Decision ) ──► [FORWARD: Filter checks POST-DNAT dst IP!]   │
│         │                                                               │
│         ▼                                                               │
│   [POSTROUTING: type nat, priority srcnat (100)]                        │
│     • SNAT / MASQUERADE                                                 │
│     • Runs AFTER Routing Decision so egress interface (oifname) is known│
│         │                                                               │
│         ▼                                                               │
│   Outbound Wire                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

> [!IMPORTANT]
> **Critical firewall interaction:** Because `PREROUTING` DNAT executes **before** `FORWARD`, your `forward` firewall rule must match the **internal (post-DNAT) destination IP and port** (e.g., `ip daddr 10.20.0.10 tcp dport 8080`), NOT the public WAN IP! Alternatively, match `ct status dnat` in `forward` to accept any connection explicitly translated by your DNAT rules.

## SNAT vs MASQUERADE (Dynamic vs Static Outbound NAT)

- **`snat to <static-ip>`:** Use whenever your WAN interface has a **static IP** (or pool of public IPs). Faster than `masquerade` because the kernel does not have to query the egress interface's primary IP on every new connection, and existing conntrack entries survive brief link flaps.
- **`masquerade`:** Use when the WAN interface gets a **dynamic IP** via DHCP/PPPoE. Automatically looks up the egress interface IP and flushes NAT entries if that IP changes.

```nft
table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;

        # Static WAN IP with randomized source port allocation (hardens against port prediction)
        oifname "wan0" ip saddr 10.10.0.0/16 snat to 198.51.100.1 random,persistent

        # Or dynamic DHCP WAN:
        # oifname "wan0" ip saddr 10.10.0.0/16 masquerade
    }
}
```

### Multi-IP Pool SNAT & CGNAT (Carrier-Grade NAT)

When tens of thousands of internal hosts share NAT, a single public IP runs out of the ~64,000 ephemeral TCP/UDP ports per destination `(dst_ip, dst_port)` tuple (port exhaustion). Spread outbound flows across a public CIDR pool or map fixed deterministic port ranges per subscriber (RFC 7422 CGNAT):

```nft
# Pool SNAT across 198.51.100.16 - 198.51.100.31 (persistent keeps same client on same external IP)
oifname "wan0" ip saddr 100.64.0.0/16 snat to 198.51.100.16-198.51.100.31 persistent

# Deterministic CGNAT port slice for subscriber 100.64.0.10 -> ports 10000-11999
oifname "wan0" ip saddr 100.64.0.10 snat to 198.51.100.16:10000-11999
```

## DNAT, Port Forwarding, and Hairpin NAT (NAT Reflection)

### Standard Inbound Port Forwarding (DNAT)

```nft
table ip nat {
    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;

        # Forward WAN:443 -> internal DMZ web server 10.20.0.10:8443
        iifname "wan0" tcp dport 443 dnat to 10.20.0.10:8443

        # Map multiple ports cleanly with a DNAT map
        iifname "wan0" tcp dport map {
            80  : 10.20.0.10,
            443 : 10.20.0.10,
            25  : 10.20.0.25
        }
    }
}
```

### The Hairpin NAT (NAT Reflection) Problem & Solution

Suppose a client on LAN (`10.10.0.50`) connects to your public WAN IP (`198.51.100.1:443`), which DNATs to a server on the **exact same LAN subnet** (`10.10.0.10:443`):

1. Without hairpin SNAT: The router rewrites Dst IP `198.51.100.1 → 10.10.0.10` and forwards it to the server — **leaving Src IP as `10.10.0.50`**.
2. The server (`10.10.0.10`) sees Src IP `10.10.0.50` is on its **own local subnet** and sends the TCP SYN-ACK **directly back to `10.10.0.50`**, bypassing the router!
3. The client (`10.10.0.50`) expected a SYN-ACK from `198.51.100.1`, not `10.10.0.10`, and drops it with a TCP RST!

**The fix:** Add a matching **POSTROUTING SNAT** rule for LAN-to-LAN traffic that underwent DNAT so replies route back through the router:

```nft
table ip nat {
    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;
        # Match destination WAN IP regardless of ingress interface (works for WAN AND LAN!)
        ip daddr 198.51.100.1 tcp dport 443 dnat to 10.10.0.10:443
    }
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        # Normal outbound WAN masquerade
        oifname "wan0" masquerade
        # Hairpin SNAT: rewrite source IP when LAN client hits DNAT'd LAN server
        iifname "lan0" oifname "lan0" ip saddr 10.10.0.0/24 ip daddr 10.10.0.10 masquerade
    }
}
```

## Stateless 1:1 NAT, Prefix Translation (NPTv6), and NAT64

- **Stateless 1:1 NAT (`ip saddr set` / `ip daddr set`):** Rewrites IP headers in `raw`/`filter` **without** allocating a single `nf_conntrack` entry — capable of multi-million PPS line-rate translation on DDoS scrubbing or high-speed interconnects:
  ```nft
  table ip stateless_nat {
      chain prerouting {
          type filter hook prerouting priority raw; policy accept;
          ip daddr 198.51.100.50 ip daddr set 10.50.0.50 notrack
      }
      chain postrouting {
          type filter hook postrouting priority raw; policy accept;
          ip saddr 10.50.0.50 ip saddr set 198.51.100.50 notrack
      }
  }
  ```
- **1:1 Prefix NAT (NETMAP):** Maps an entire `/24` onto another `/24` preserving host bits (essential when two merged companies have overlapping `10.0.0.0/24` VPN subnets!):
  ```nft
  snat ip prefix to ip saddr map { 10.0.0.0/24 : 10.200.0.0/24 }
  ```
- **NAT64 / DNS64 (IPv6-only clients reaching IPv4 Internet):** Linux pairs **Jool** (kernel-module stateful/stateless RFC 6146 NAT64 using `64:ff9b::/96`) with **Unbound/BIND DNS64** to synthesize `AAAA` records from `A` records seamlessly.

## Debugging NAT & Connection Tracking with `conntrack`

`conntrack` shows you the exact original tuple AND reply tuple side-by-side — making NAT debugging effortless:

```bash
# List all active NAT translations (look for mismatched original src vs reply dst!)
sudo conntrack -L --src-nat
sudo conntrack -L --dst-nat

# Stream NAT connection creation/teardown events in real time
sudo conntrack -E

# Delete a stuck NAT entry (e.g., stale UDP SIP/WireGuard flow after WAN IP change!)
sudo conntrack -D -p udp --dport 51820

# Inspect conntrack table utilization and drop/insert_failed counters
sudo conntrack -S
```

Reading a `conntrack -L` line:

```text
tcp  6 431990 ESTABLISHED src=10.10.0.50 dst=93.184.216.34 sport=51234 dport=443 \
                          src=93.184.216.34 dst=198.51.100.1 sport=443 dport=51234 [ASSURED]
```

Notice the **original direction** (`src=10.10.0.50 → dst=93.184.216.34`) vs the **expected reply direction** (`src=93.184.216.34 → dst=198.51.100.1`). Because `reply dst` (`198.51.100.1`) differs from `original src` (`10.10.0.50`), you know immediately that **SNAT** occurred!

## Hands-On Lab 12: Full SNAT, DNAT, and Hairpin NAT Verification Lab

Build a LAN (`client` + `web`), a NAT gateway (`gw`), and a simulated Internet host (`inet`), verify outbound SNAT, inbound DNAT port forwarding, and fix Hairpin NAT so `client` can reach `web` via `gw`'s public IP!

```bash
# 1. Create namespaces: client, web, lan-sw, gw, inet
for ns in client web lan-sw gw inet; do sudo ip netns add "$ns"; done

# 2. Wire client + web + gw-lan onto lan-sw; wire gw-wan <-> inet
sudo ip link add cli0 type veth peer name sw-cli
sudo ip link add web0 type veth peer name sw-web
sudo ip link add gw-lan type veth peer name sw-gw
sudo ip link add gw-wan type veth peer name inet0

sudo ip link set cli0 netns client; sudo ip link set sw-cli netns lan-sw
sudo ip link set web0 netns web;    sudo ip link set sw-web netns lan-sw
sudo ip link set gw-lan netns gw;   sudo ip link set sw-gw netns lan-sw
sudo ip link set gw-wan netns gw;   sudo ip link set inet0 netns inet

# 3. Configure LAN bridge
sudo ip -n lan-sw link add br0 type bridge
sudo ip -n lan-sw link set br0 up
for p in sw-cli sw-web sw-gw; do sudo ip -n lan-sw link set "$p" master br0 up; done

# 4. Configure IPs (NOTICE: 'inet' has NO route back to RFC 1918 10.10.0.0/24!)
sudo ip -n client link set cli0 up; sudo ip -n client addr add 10.10.0.50/24 dev cli0
sudo ip -n client route add default via 10.10.0.1

sudo ip -n web link set web0 up; sudo ip -n web addr add 10.10.0.10/24 dev web0
sudo ip -n web route add default via 10.10.0.1

sudo ip -n gw link set gw-lan up; sudo ip -n gw addr add 10.10.0.1/24 dev gw-lan
sudo ip -n gw link set gw-wan up; sudo ip -n gw addr add 198.51.100.1/24 dev gw-wan
sudo ip netns exec gw sysctl -w net.ipv4.ip_forward=1

sudo ip -n inet link set inet0 up; sudo ip -n inet addr add 198.51.100.200/24 dev inet0

# 5. Start web server inside 'web' on port 8080
sudo ip netns exec web python3 -m http.server 8080 --bind 10.10.0.10 >/dev/null 2>&1 &
WEB_PID=$!
sleep 0.5

# 6. Apply SNAT + DNAT + Hairpin NAT ruleset on gw
sudo ip netns exec gw nft -f - << 'EOF'
flush ruleset
table ip nat {
    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;
        # DNAT public IP 198.51.100.1:80 -> internal web 10.10.0.10:8080
        ip daddr 198.51.100.1 tcp dport 80 dnat to 10.10.0.10:8080
    }
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        # Outbound SNAT for LAN -> Internet
        oifname "gw-wan" ip saddr 10.10.0.0/24 snat to 198.51.100.1
        # Hairpin SNAT for LAN client -> public IP -> LAN web server
        iifname "gw-lan" oifname "gw-lan" ip saddr 10.10.0.0/24 ip daddr 10.10.0.10 snat to 10.10.0.1
    }
}
EOF

# 7. Test 1: Outbound SNAT (client -> inet succeeds even though inet has NO route to 10.10.0.0/24!)
echo "=== Test 1: Outbound SNAT (client -> 198.51.100.200) ==="
sudo ip netns exec client ping -c 2 198.51.100.200

# 8. Test 2: Inbound DNAT (inet -> 198.51.100.1:80 reaches web 10.10.0.10:8080)
echo "=== Test 2: Inbound DNAT (inet -> 198.51.100.1:80) ==="
sudo ip netns exec inet curl -sI http://198.51.100.1:80 | head -n 1

# 9. Test 3: Hairpin NAT (LAN client -> public IP 198.51.100.1:80 reaches LAN web!)
echo "=== Test 3: Hairpin NAT (client -> 198.51.100.1:80) ==="
sudo ip netns exec client curl -sI http://198.51.100.1:80 | head -n 1

# 10. Inspect conntrack translation table on gw
sudo ip netns exec gw conntrack -L 2>/dev/null || true

# 11. Cleanup
sudo kill "$WEB_PID" 2>/dev/null || true
for ns in client web lan-sw gw inet; do sudo ip netns del "$ns"; done
```

**What you verified:** You proved outbound SNAT hides RFC 1918 addresses from `inet`, inbound DNAT maps public port 80 to internal `10.10.0.10:8080`, and Hairpin SNAT makes the same public URL work cleanly from inside the LAN.
