---
title: 'Chapter 16: Capstone Lab: Enterprise Linux Network Fabric'
---

# Chapter 16: Capstone Lab: Enterprise Linux Network Fabric

Every previous chapter isolated one subsystem so you could master its kernel mechanics without noise. Real enterprise Linux networks run **all of them together simultaneously**.

In this capstone, you will construct a complete, self-contained **Multi-Site Enterprise Linux Network Fabric** inside Linux network namespaces combining:
- **IEEE 802.3ad (LACP) Bonding** between the core switch and edge firewall/router
- **VLAN-Aware Linux Switching** (`vlan_filtering=1`) isolating **VLAN 10 (Corp LAN)** and **VLAN 20 (DMZ)** over an 802.1Q trunk atop the LACP bond
- **DHCP & Split-DNS Services** (`dnsmasq`) dynamically serving Corp LAN clients and resolving internal hostnames
- **Stateful Zone-Based `nftables` Firewall + SNAT + DNAT** at the Site A Edge Gateway
- **Encrypted Site-to-Site WireGuard VPN** connecting Site A to a remote Site B across a simulated untrusted Internet (`isp`)
- **Automated Verification & Fault-Injection Drills** proving LACP failover, firewall enforcement, NAT translation, DNS resolution, and VPN encryption end to end

## Capstone Topology Blueprint

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                     ENTERPRISE LINUX NETWORK FABRIC                         │
│                                                                             │
│   [corp-pc] (DHCP, VLAN 10)        [dmz-web] (10.20.0.10:8080, VLAN 20)     │
│        │ (access VID 10)                │ (access VID 20)                   │
│        └──────────────┬─────────────────┘                                   │
│                       ▼                                                     │
│          [ core-sw: br0 (vlan_filtering=1) ]                                │
│                       │                                                     │
│            ═══[ bond0 (802.3ad LACP: 2 links) ]═══  (802.1Q Trunk: 10, 20)  │
│                       │                                                     │
│                       ▼                                                     │
│   [ edge-a: bond0.10 (10.10.0.1) | bond0.20 (10.20.0.1) ]                   │
│     • DHCP + DNS Server (dnsmasq on bond0.10)                               │
│     • Stateful Zone Firewall (nftables vmap) + SNAT + DNAT                  │
│     • WireGuard Tunnel (wg0: 10.250.0.1/30)                                 │
│                       │ (wan: 198.51.100.2/30)                              │
│                       ▼                                                     │
│              [ isp (Untrusted Internet) ]                                   │
│               198.51.100.1 ── 203.0.113.1                                   │
│                  │                │                                         │
│      (inet-client: 198.51.100.10) ▼ (wan: 203.0.113.2/30)                   │
│                         [ edge-b: wg0 (10.250.0.2/30) ]                     │
│                                   │ (lan: 10.30.0.1/24)                     │
│                                   ▼                                         │
│                       [ branch-app (10.30.0.50/24) ]                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Addressing & Policy Plan

| Zone / Link | Subnet / VLAN | Key Hosts & Services |
|---|---|---|
| **Site A Corp LAN** | `10.10.0.0/24` (VLAN 10) | `edge-a` (`10.10.0.1`, DHCP/DNS), `corp-pc` (DHCP `10.10.0.100–150`) |
| **Site A DMZ** | `10.20.0.0/24` (VLAN 20) | `edge-a` (`10.20.0.1`), `dmz-web` (`10.20.0.10`, HTTP `:8080`) |
| **Site A ↔ Core Trunk** | 802.3ad LACP (`bond0`) | Two parallel links (`lacp0`, `lacp1`) carrying VLAN 10 & 20 tagged |
| **Site A WAN / ISP** | `198.51.100.0/24` | `edge-a` (`198.51.100.2`), `isp` (`198.51.100.1`), `inet-cli` (`198.51.100.10`) |
| **Site B WAN / ISP** | `203.0.113.0/24` | `isp` (`203.0.113.1`), `edge-b` (`203.0.113.2`) |
| **WireGuard Overlay** | `10.250.0.0/30` (UDP 51820) | `edge-a` (`10.250.0.1`) ↔ `edge-b` (`10.250.0.2`) |
| **Site B Branch LAN** | `10.30.0.0/24` | `edge-b` (`10.30.0.1`), `branch-app` (`10.30.0.50`, HTTP `:9000`) |

### Security Policy Enforced on `edge-a`

1. **Corp LAN → WAN:** Permitted + SNAT (`198.51.100.2`).
2. **Corp LAN → DMZ & Site B VPN (`wg0`):** Permitted (no NAT over VPN — native private routing!).
3. **WAN → DMZ:** Blocked by default **except** public DNAT `198.51.100.2:80 → 10.20.0.10:8080`.
4. **DMZ → Corp LAN & Site B VPN:** **Strictly denied** (lateral-movement containment).

## Complete Reproducible Capstone Build Script

Save this script as `capstone-fabric.sh`, run `sudo bash capstone-fabric.sh up` to deploy the entire enterprise fabric, `test` to execute the verification suite, and `down` to tear everything down cleanly.

```bash
#!/usr/bin/env bash
set -euo pipefail

LAB_DIR="/tmp/boring-linux-capstone"
NAMESPACES=(corp-pc dmz-web core-sw edge-a isp inet-cli edge-b branch-app)

cleanup() {
    echo "[*] Tearing down capstone fabric..."
    if [[ -d "$LAB_DIR" ]]; then
        for pidfile in "$LAB_DIR"/*.pid; do
            [[ -f "$pidfile" ]] && kill "$(cat "$pidfile")" 2>/dev/null || true
        done
        rm -rf "$LAB_DIR"
    fi
    for ns in "${NAMESPACES[@]}"; do
        ip netns del "$ns" 2>/dev/null || true
    done
    echo "[+] Cleanup complete."
}

deploy() {
    cleanup
    mkdir -p "$LAB_DIR"
    chmod 700 "$LAB_DIR"

    echo "[*] Creating network namespaces..."
    for ns in "${NAMESPACES[@]}"; do
        ip netns add "$ns"
        ip -n "$ns" link set lo up
    done

    echo "[*] Wiring virtual patch cables..."
    # Access ports into core-sw
    ip link add corp0 type veth peer name sw-corp
    ip link add dmz0  type veth peer name sw-dmz
    # Dual LACP trunk cables between core-sw and edge-a
    ip link add sw-lacp0 type veth peer name ea-lacp0
    ip link add sw-lacp1 type veth peer name ea-lacp1
    # ISP fabric (bridge inside isp so inet-cli shares Site A public WAN segment)
    ip link add ea-wan   type veth peer name isp-ea
    ip link add icli0    type veth peer name isp-icli
    ip link add isp-eb   type veth peer name eb-wan
    # Site B branch LAN
    ip link add eb-lan   type veth peer name br0

    ip link set corp0    netns corp-pc;    ip link set sw-corp  netns core-sw
    ip link set dmz0     netns dmz-web;    ip link set sw-dmz   netns core-sw
    ip link set sw-lacp0 netns core-sw;    ip link set ea-lacp0 netns edge-a
    ip link set sw-lacp1 netns core-sw;    ip link set ea-lacp1 netns edge-a
    ip link set ea-wan   netns edge-a;     ip link set isp-ea   netns isp
    ip link set icli0    netns inet-cli;   ip link set isp-icli netns isp
    ip link set isp-eb   netns isp;        ip link set eb-wan   netns edge-b
    ip link set eb-lan   netns edge-b;     ip link set br0      netns branch-app

    echo "[*] Configuring 802.3ad LACP bond + VLAN-aware switch on core-sw..."
    ip -n core-sw link add bond0 type bond mode 802.3ad miimon 100 lacp_rate fast xmit_hash_policy layer3+4
    ip -n core-sw link set sw-lacp0 down; ip -n core-sw link set sw-lacp1 down
    ip -n core-sw link set sw-lacp0 master bond0
    ip -n core-sw link set sw-lacp1 master bond0
    ip -n core-sw link set bond0 up

    ip -n core-sw link add br0 type bridge vlan_filtering 1 vlan_default_pvid 0
    ip -n core-sw link set br0 up
    for p in sw-corp sw-dmz bond0; do ip -n core-sw link set "$p" master br0 up; done
    ip netns exec core-sw bridge vlan add vid 10 pvid untagged dev sw-corp
    ip netns exec core-sw bridge vlan add vid 20 pvid untagged dev sw-dmz
    ip netns exec core-sw bridge vlan add vid 10 dev bond0
    ip netns exec core-sw bridge vlan add vid 20 dev bond0

    echo "[*] Configuring edge-a LACP bond + 802.1Q subinterfaces + WAN..."
    ip -n edge-a link add bond0 type bond mode 802.3ad miimon 100 lacp_rate fast xmit_hash_policy layer3+4
    ip -n edge-a link set ea-lacp0 down; ip -n edge-a link set ea-lacp1 down
    ip -n edge-a link set ea-lacp0 master bond0
    ip -n edge-a link set ea-lacp1 master bond0
    ip -n edge-a link set bond0 up

    ip -n edge-a link add link bond0 name bond0.10 type vlan id 10
    ip -n edge-a link add link bond0 name bond0.20 type vlan id 20
    ip -n edge-a addr add 10.10.0.1/24 dev bond0.10
    ip -n edge-a addr add 10.20.0.1/24 dev bond0.20
    ip -n edge-a link set bond0.10 up
    ip -n edge-a link set bond0.20 up

    ip -n edge-a link set ea-wan up
    ip -n edge-a addr add 198.51.100.2/24 dev ea-wan
    ip -n edge-a route add default via 198.51.100.1
    ip netns exec edge-a sysctl -qw net.ipv4.ip_forward=1

    echo "[*] Configuring simulated ISP + Internet client + Site B..."
    ip -n isp link add br-wan-a type bridge
    ip -n isp link set br-wan-a up
    ip -n isp link set isp-ea   master br-wan-a up
    ip -n isp link set isp-icli master br-wan-a up
    ip -n isp addr add 198.51.100.1/24 dev br-wan-a
    ip -n isp link set isp-eb up
    ip -n isp addr add 203.0.113.1/24 dev isp-eb
    ip netns exec isp sysctl -qw net.ipv4.ip_forward=1

    ip -n inet-cli link set icli0 up
    ip -n inet-cli addr add 198.51.100.10/24 dev icli0
    ip -n inet-cli route add default via 198.51.100.1

    ip -n edge-b link set eb-wan up
    ip -n edge-b addr add 203.0.113.2/24 dev eb-wan
    ip -n edge-b route add default via 203.0.113.1
    ip -n edge-b link set eb-lan up
    ip -n edge-b addr add 10.30.0.1/24 dev eb-lan
    ip netns exec edge-b sysctl -qw net.ipv4.ip_forward=1

    ip -n branch-app link set br0 up
    ip -n branch-app addr add 10.30.0.50/24 dev br0
    ip -n branch-app route add default via 10.30.0.1

    echo "[*] Configuring DMZ web server + Branch app server..."
    ip -n dmz-web link set dmz0 up
    ip -n dmz-web addr add 10.20.0.10/24 dev dmz0
    ip -n dmz-web route add default via 10.20.0.1
    ip netns exec dmz-web python3 -m http.server 8080 --bind 10.20.0.10 >/dev/null 2>&1 &
    echo $! > "$LAB_DIR/dmz-web.pid"

    ip netns exec branch-app python3 -m http.server 9000 --bind 10.30.0.50 >/dev/null 2>&1 &
    echo $! > "$LAB_DIR/branch-app.pid"

    echo "[*] Configuring encrypted WireGuard site-to-site VPN (edge-a <-> edge-b)..."
    wg genkey | tee "$LAB_DIR/a.priv" | wg pubkey > "$LAB_DIR/a.pub"
    wg genkey | tee "$LAB_DIR/b.priv" | wg pubkey > "$LAB_DIR/b.pub"
    PUB_A=$(cat "$LAB_DIR/a.pub")
    PUB_B=$(cat "$LAB_DIR/b.pub")

    ip -n edge-a link add wg0 type wireguard
    ip -n edge-a addr add 10.250.0.1/30 dev wg0
    ip netns exec edge-a wg set wg0 listen-port 51820 private-key "$LAB_DIR/a.priv" \
        peer "$PUB_B" endpoint 203.0.113.2:51820 allowed-ips 10.250.0.2/32,10.30.0.0/24
    ip -n edge-a link set wg0 mtu 1420 up
    ip -n edge-a route add 10.30.0.0/24 dev wg0

    ip -n edge-b link add wg0 type wireguard
    ip -n edge-b addr add 10.250.0.2/30 dev wg0
    ip netns exec edge-b wg set wg0 listen-port 51820 private-key "$LAB_DIR/b.priv" \
        peer "$PUB_A" endpoint 198.51.100.2:51820 allowed-ips 10.250.0.1/32,10.10.0.0/24
    ip -n edge-b link set wg0 mtu 1420 up
    ip -n edge-b route add 10.10.0.0/24 dev wg0

    echo "[*] Loading stateful Zone Firewall + SNAT + DNAT on edge-a..."
    ip netns exec edge-a nft -f - << 'EOF'
flush ruleset
table inet edge_fw {
    chain input {
        type filter hook input priority filter; policy drop;
        iifname "lo" accept
        ct state established,related accept
        ct state invalid drop
        ip protocol icmp accept
        iifname "bond0.10" udp dport { 53, 67 } accept
        iifname "bond0.10" tcp dport 53 accept
        iifname "ea-wan" udp dport 51820 accept
        counter drop
    }
    chain forward {
        type filter hook forward priority filter; policy drop;
        ct state established,related accept
        ct state invalid drop
        # MSS clamp for WireGuard overlay
        oifname "wg0" tcp flags syn / syn,rst tcp option maxseg size set rt mtu
        iifname . oifname vmap {
            "bond0.10" . "ea-wan"   : accept,
            "bond0.10" . "bond0.20" : accept,
            "bond0.10" . "wg0"      : accept,
            "wg0"      . "bond0.10" : accept,
            "ea-wan"   . "bond0.20" : jump wan_to_dmz
        }
        counter drop
    }
    chain wan_to_dmz {
        ip daddr 10.20.0.10 tcp dport 8080 counter accept
    }
    chain output {
        type filter hook output priority filter; policy accept;
    }
}
table ip edge_nat {
    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;
        iifname "ea-wan" ip daddr 198.51.100.2 tcp dport 80 dnat to 10.20.0.10:8080
    }
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        oifname "ea-wan" ip saddr 10.10.0.0/24 snat to 198.51.100.2
    }
}
EOF

    echo "[*] Starting DHCP + Internal DNS (dnsmasq) on edge-a VLAN 10..."
    ip netns exec edge-a dnsmasq \
        --interface=bond0.10 --bind-interfaces --no-resolv --no-hosts \
        --dhcp-range=10.10.0.100,10.10.0.150,255.255.255.0,1h \
        --dhcp-option=option:router,10.10.0.1 \
        --dhcp-option=option:dns-server,10.10.0.1 \
        --address=/web.dmz.corp/10.20.0.10 \
        --address=/app.branch.corp/10.30.0.50 \
        --dhcp-leasefile="$LAB_DIR/dhcp.leases" \
        --pid-file="$LAB_DIR/dnsmasq.pid"

    echo "[*] Waiting 2s for 802.3ad LACP aggregation, then provisioning corp-pc via DHCP..."
    sleep 2
    ip -n corp-pc link set corp0 up
    if command -v dhclient >/dev/null 2>&1; then
        ip netns exec corp-pc dhclient -1 -lf "$LAB_DIR/corp.lease" -pf "$LAB_DIR/corp-dhclient.pid" corp0
    elif command -v dhcpcd >/dev/null 2>&1; then
        ip netns exec corp-pc dhcpcd -1 corp0
    else
        # Fallback if neither DHCP client binary is installed
        ip -n corp-pc addr add 10.10.0.100/24 dev corp0
        ip -n corp-pc route add default via 10.10.0.1
    fi

    echo "[+] Enterprise Linux Network Fabric is UP!"
}

verify() {
    echo "=== 1. LACP Bond Status (edge-a <-> core-sw) ==="
    ip netns exec edge-a grep -E 'Bonding Mode|Slave Interface|MII Status|Number of ports' /proc/net/bonding/bond0

    echo "=== 2. Corp-PC DHCP IP & Internal DNS Resolution ==="
    ip -n corp-pc -br addr show dev corp0
    ip netns exec corp-pc dig @10.10.0.1 web.dmz.corp +short
    ip netns exec corp-pc dig @10.10.0.1 app.branch.corp +short

    echo "=== 3. Corp-PC -> Internet SNAT (198.51.100.10) ==="
    ip netns exec corp-pc ping -c 2 198.51.100.10

    echo "=== 4. Internet -> DMZ Public DNAT (198.51.100.2:80 -> 10.20.0.10:8080) ==="
    ip netns exec inet-cli curl -sI http://198.51.100.2:80 | head -n 1

    echo "=== 5. Corp-PC -> Site B Branch App over Encrypted WireGuard VPN ==="
    ip netns exec corp-pc curl -sI http://10.30.0.50:9000 | head -n 1
    ip netns exec edge-a wg show

    echo "=== 6. Security Isolation: DMZ -> Corp LAN MUST BE BLOCKED ==="
    if ip netns exec dmz-web ping -c 1 -W 1 10.10.0.1 >/dev/null 2>&1; then
        echo "FAIL: DMZ reached Corp LAN!" && exit 1
    else
        echo "PASS: DMZ -> Corp LAN blocked by nftables zone policy."
    fi

    echo "=== 7. Live LACP Failover Drill (Cutting ea-lacp0 during traffic) ==="
    ip -n edge-a link set ea-lacp0 down
    ip netns exec corp-pc ping -c 3 10.30.0.50
    ip -n edge-a link set ea-lacp0 up
    echo "[+] ALL CAPSTONE CHECKS PASSED!"
}

case "${1:-}" in
    up)   deploy ;;
    test) verify ;;
    down) cleanup ;;
    *)    echo "Usage: sudo $0 {up|test|down}" && exit 1 ;;
esac
```

## Fault-Injection Drills to Master This Fabric

Once `sudo bash capstone-fabric.sh test` passes cleanly, practice breaking and diagnosing your own fabric using the tools from [Chapter 15](15-network-troubleshooting-packet-forensics-and-observability.md):

1. **VLAN Trunk Misconfiguration:** Remove VLAN 20 from `bond0` on `core-sw` (`sudo ip netns exec core-sw bridge vlan del vid 20 dev bond0`) and observe how `corp-pc` keeps working while public DNAT to `dmz-web` fails silently. Diagnose using `bridge -c vlan show` and `tcpdump -e`.
2. **WireGuard `AllowedIPs` Drop:** Remove `10.30.0.0/24` from `edge-a`'s WireGuard peer config and watch `ping` return `Required key not available` (`ENOKEY`) — the unmistakable signature of a WireGuard Cryptokey Routing mismatch.
3. **Strict `rp_filter` Breakage:** Change routing asymmetrically or inspect `nft monitor trace` on `edge-a` while `dmz-web` tries to probe `10.10.0.100`, proving the exact `vmap` fallback drop rule.
4. **WAN Latency & Jitter Simulation:** Inject `tc qdisc replace dev isp-eb root netem delay 60ms 10ms loss 2%` inside `isp` and measure how WireGuard + TCP behaves under real wide-area impairment with `iperf3` and `ss -tin`.

## Closing Synthesis

You now hold a complete, end-to-end engineering map of networking on Linux:
- **Layer 1 & 2:** Hardware rings, NAPI, `ethtool`, predictable naming, ARP/NDP neighbor state machines, 802.1Q/QinQ VLANs, VLAN-aware bridges, STP, and 802.3ad LACP bonds.
- **Layer 3 & Services:** Kernel `sk_buff` forwarding, `rp_filter`, PMTUD, ECMP nexthops, RPDB Policy-Based Routing, VRFs, FRRouting (BGP/OSPF/BFD), ISC Kea / `dnsmasq` DHCP, and `systemd-resolved` / Unbound / BIND 9 DNS.
- **Security, Overlays & Forensics:** Netfilter hooks, `conntrack`, `nftables` sets/vmaps/flowtables, SNAT/DNAT/Hairpin NAT, WireGuard, strongSwan IPsec/XFRM, VXLAN, Open vSwitch, `tc` QoS, eBPF/XDP, and kernel-level drop tracing.

Linux is no longer a black box between your applications and the wire — it is a transparent, scriptable, carrier-grade network operating system you can build, automate, and debug from silicon to socket.
