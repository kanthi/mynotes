---
title: 'Chapter 15: Network Troubleshooting, Packet Forensics, and Observability'
---

# Chapter 15: Network Troubleshooting, Packet Forensics, and Observability

When production traffic stops flowing, guessing wastes precious minutes. Senior Linux network engineers rely on a disciplined **bottom-up (Layer 1 → Layer 7) or divide-and-conquer** diagnostic workflow backed by live kernel counters, socket inspection (`ss`), precision packet captures (`tcpdump`/`tshark`), and eBPF/tracepoint drop hunters (`dropwatch`, `pwru`, `nft monitor trace`) that tell you the *exact* kernel function where a packet died.

## The Structured Linux L1–L7 Diagnostic Checklist

| Layer | Question to Answer | Primary Linux Commands |
|---|---|---|
| **L1 Physical / Driver** | Is carrier up? Any CRC, ring, or optic errors? | `ip -br -c link`, `ethtool <dev>`, `ethtool -S <dev>`, `dmesg \| grep -iE 'link\|eth\|enp'` |
| **L2 Ethernet / VLAN / Bridge** | Right MAC, VLAN tag, FDB port, LACP aggregator, ARP/NDP state? | `ip -c neigh`, `bridge vlan show`, `bridge fdb show`, `cat /proc/net/bonding/bond0`, `lldpcli show neighbors` |
| **L3 IP & Routing** | Right IP/mask? Which route/rule/VRF wins? Is `ip_forward` or `rp_filter` dropping it? | `ip -br -c addr`, `ip route get <dst>`, `ip rule show`, `sysctl net.ipv4.ip_forward net.ipv4.conf.all.rp_filter` |
| **L4 Transport & Sockets** | Is the daemon listening on the right IP/port? Any SYN backlog drops, retransmits, or TIME_WAIT exhaustion? | `ss -tunap`, `ss -tin`, `nstat -az`, `cat /proc/net/softnet_stat` |
| **Firewall / NAT / VPN** | Is `nftables`, `conntrack`, or XFRM/WireGuard dropping or mis-translating the flow? | `nft list ruleset`, `nft monitor trace`, `conntrack -L`, `wg show`, `ip -s xfrm state` |
| **L7 DNS & Application** | Does DNS resolve identically in NSS and `dig`? Does TLS/HTTP respond? | `getent ahosts <name>`, `resolvectl query <name>`, `dig +trace`, `curl -vvv --resolve ...` |

## Socket and Stack Inspection: `ss`, `nstat`, and `softnet_stat`

### Mastering `ss` (Socket Statistics)

`ss` talks directly to the kernel via `inet_diag` Netlink — orders of magnitude faster than legacy `netstat`:

```bash
# List all listening TCP/UDP sockets with owning PID/process
sudo ss -tulnp

# Show TCP internal state: congestion control, RTT, cwnd, retransmits, pacing rate, socket queues!
ss -tinp state established '( dport = :443 or sport = :443 )'

# Summary counts by socket state (spot SYN floods or CLOSE_WAIT app leaks instantly)
ss -s

# Check listen backlog saturation: Recv-Q == Send-Q means the app cannot accept() fast enough!
ss -lnt
```

> [!IMPORTANT]
> **Reading `ss -lnt` (Listening Sockets):** For a **LISTEN** socket, `Recv-Q` is the current number of completed connections waiting for `accept()`, and `Send-Q` is the maximum listen backlog (`min(backlog, net.core.somaxconn)`). If `Recv-Q` approaches `Send-Q`, the kernel starts dropping new SYNs (`TcpExtListenOverflows` / `TcpExtListenDrops`)!

### Reading Kernel SNMP/Stack Counters with `nstat`

`nstat` tracks deltas in `/proc/net/snmp` and `/proc/net/netstat` cleanly:

```bash
# Reset baseline, wait 5 seconds, print ONLY counters that incremented during the fault!
nstat -r >/dev/null
sleep 5
nstat -az | grep -iE 'Drop|Overflow|Retrans|Abort|Err|OutOfWindow|Listen|Syncookies|Martian'
```

### Detecting Softirq Packet Drops (`/proc/net/softnet_stat`)

Each line in `/proc/net/softnet_stat` represents one CPU core (all values in hexadecimal):

- **Column 1:** Total packets processed by this CPU's NAPI poll loop.
- **Column 2 (`dropped`):** Packets dropped because `net.core.netdev_max_backlog` overflowed!
- **Column 3 (`time_squeeze`):** NAPI ran out of CPU budget (`netdev_budget` / `netdev_budget_usecs`) before draining the RX ring!

```bash
# Pretty-print softnet_stat in decimal per CPU core
awk '{printf "CPU%02d: total=%u dropped=%u time_squeeze=%u\n", NR-1, strtonum("0x"$1), strtonum("0x"$2), strtonum("0x"$3)}' /proc/net/softnet_stat
```

## Packet Capture Mastery: `tcpdump` and `tshark`

### Essential `tcpdump` Recipes

Always pass **`-nn`** (no DNS/port name lookups — prevents hangs and hides nothing) and **`-i any`** when you aren't sure which interface receives the packet:

```bash
# Capture Ethernet headers (-e), VLAN tags, and exact timestamps (-ttt) without truncation (-s0)
sudo tcpdump -i eth0 -nn -e -vvv -s0 'host 10.10.0.50 and (port 443 or icmp)'

# Capture ONLY TCP SYN, FIN, and RST packets (instant connection handshake & reset triage!)
sudo tcpdump -i any -nn 'tcp[tcpflags] & (tcp-syn|tcp-fin|tcp-rst) != 0'

# Capture DNS queries/answers cleanly
sudo tcpdump -i any -nn -v 'udp port 53 or tcp port 53'

# Rotating ring-buffer capture for intermittent production bugs (10 files x 100 MB each)
sudo tcpdump -i eth0 -s0 -W 10 -C 100 -w /var/tmp/cap.pcap

# Remote live Wireshark capture over SSH without storing a pcap on the server!
ssh root@linux-router "tcpdump -i eth0 -U -s0 -w - 'not port 22'" | wireshark -k -i -
```

> [!WARNING]
> **BPF filter trap with VLAN tags:** A filter like `'tcp port 80'` does **NOT** match 802.1Q VLAN-tagged frames unless you explicitly write `'vlan and tcp port 80'` (or `'(tcp port 80) or (vlan and tcp port 80)'`), because the 4-byte VLAN tag shifts all subsequent header offsets!

### Deep Analysis with `tshark`

```bash
# Find top TCP retransmissions and out-of-order frames in a pcap
tshark -r /var/tmp/cap.pcap -Y 'tcp.analysis.retransmission || tcp.analysis.lost_segment'

# TCP conversation table sorted by bytes transferred
tshark -r /var/tmp/cap.pcap -q -z conv,tcp

# Extract TLS SNI hostnames and HTTP response latencies from a capture
tshark -r /var/tmp/cap.pcap -Y 'tls.handshake.extensions_server_name' -T fields -e ip.src -e tls.handshake.extensions_server_name
```

## Active Path & Throughput Testing: `mtr`, `tracepath`, and `iperf3`

```bash
# Combined continuous traceroute + packet-loss & jitter statistics (TCP SYN mode bypasses ICMP blocks!)
mtr -n -T -P 443 --report --report-cycles 30 1.1.1.1

# Discover exact Path MTU hop-by-hop without root privileges
tracepath -n 198.51.100.10

# Bidirectional multi-stream throughput test (isolates single-flow TCP BDP vs LACP hash limits)
# Server: iperf3 -s
# Client: 8 parallel streams, 10 seconds, reverse direction (-R) tests download
iperf3 -c 10.10.0.20 -P 8 -t 10
iperf3 -c 10.10.0.20 -u -b 1G -t 10   # UDP jitter & packet loss measurement
```

## Kernel Drop Hunting: `dropwatch`, `perf`, and `pwru` (eBPF)

When `tcpdump` shows a packet **entering** `eth0`, `nftables` shows zero drops, and the packet simply **vanishes inside the Linux kernel**, use kernel drop tracing — Linux kernels 5.17+ include explicit **`enum skb_drop_reason`** codes (`SKB_DROP_REASON_IP_RPFILTER`, `SKB_DROP_REASON_TCP_CSUM`, `SKB_DROP_REASON_SOCKET_RCVBUFF`, `SKB_DROP_REASON_NETFILTER_DROP`, etc.) directly inside the `skb:kfree_skb` tracepoint!

### Method 1: Instant Drop Reasons via `perf` (Zero Extra Tools Needed!)

```bash
# Record every kernel skb drop with its exact kernel drop reason for 5 seconds
sudo perf record -e skb:kfree_skb -a -- sleep 5
sudo perf script | head -n 30
```

### Method 2: `dropwatch` (Live Drop Counter Aggregator)

```bash
sudo dropwatch -l kas
# At the dropwatch prompt:
start
# Shows exact kernel symbol + drop rate, e.g., "142 drops at nf_hook_slow+0x44 [reason: NETFILTER_DROP]"
```

### Method 3: `pwru` ("Packet, Where Are You?" — Cilium eBPF Tracer)

`pwru` attaches eBPF kprobes to every kernel function taking `struct sk_buff *` and traces a single filtered packet through every single kernel C function call until delivery or drop:

```bash
sudo pwru --filter-dst-ip 10.20.0.10 --filter-dst-port 443 --output-tuple --output-meta
```

## Eight Classic Real-World Linux Network Break-Fix Scenarios

| # | Symptom | Diagnostic Signal | Root Cause & Fix |
|---|---|---|---|
| **1** | Small pings work, SSH/HTTPS hangs after TLS ClientHello | `ping -M do -s 1472` fails; `tcpdump` shows large retransmits | **MTU black hole** (VPN/VXLAN/PPPoE MTU mismatch or blocked ICMP Type 3/4) → fix MTU symmetry + TCP MSS clamping |
| **2** | Replies arrive on `eth1` in `tcpdump` but never reach the app | `nstat` shows `IPReversePathFilter` rising; `dmesg` shows martians | **Strict `rp_filter=1`** dropping asymmetric multi-WAN traffic → set `rp_filter=2` (loose) + PBR `ip rule` |
| **3** | Intermittent 5-second stalls or dropped new connections under load | `dmesg`: `nf_conntrack: table full` or `neighbor table overflow` | **`nf_conntrack_max`** or **`gc_thresh3`** too low → scale sysctls & hash buckets |
| **4** | Host pings router fine, cannot reach remote subnets behind router | `sysctl net.ipv4.ip_forward` is `0` | **Forwarding disabled** on Linux router → set `net.ipv4.ip_forward=1` |
| **5** | DNAT port forward arrives on WAN, never reaches DMZ server | `nft monitor trace` stops at `forward` drop | **Firewall checked pre-DNAT IP** instead of post-DNAT internal IP (or missing `ct status dnat accept`) |
| **6** | LAN host cannot reach its own DNAT'd service via public IP | `tcpdump` on server shows SYN-ACK going directly to client, followed by RST | **Missing Hairpin SNAT** in `postrouting` for LAN→LAN DNAT flows |
| **7** | LACP bond only uses 50% capacity or drops half the hosts | `/proc/net/bonding/bond0` shows different `Aggregator ID`s or `layer2` hash | **Switch LAG mismatch** or default `layer2` hash sending all gateway traffic down 1 link → set `layer3+4` + match switch LACP |
| **8** | `curl` fails with DNS timeout while `dig @1.1.1.1` works | `resolvectl status` shows broken VPN DNS or stale `/etc/resolv.conf` | **`systemd-resolved` split-DNS / NSS mismatch** → fix `resolvectl domain` / `/etc/nsswitch.conf` |

## Hands-On Lab 15: Intentional Multi-Fault Break-Fix Diagnostic Drill

This lab builds a broken 3-hop topology (`client <-> router <-> server`) with **three hidden simultaneous faults** (disabled forwarding, strict `rp_filter` / bad route, and a silent `nftables` drop). Practice isolating and proving each fault methodically with `ip route get`, `tcpdump`, `nft monitor trace`, and `ss`!

```bash
# === SETUP BROKEN TOPOLOGY ===
for ns in cli rtr srv; do sudo ip netns add "$ns"; done
sudo ip link add c0 type veth peer name r-c
sudo ip link add r-s type veth peer name s0
sudo ip link set c0 netns cli; sudo ip link set r-c netns rtr
sudo ip link set r-s netns rtr; sudo ip link set s0 netns srv

sudo ip -n cli link set c0 up; sudo ip -n cli addr add 10.15.1.10/24 dev c0
sudo ip -n cli route add default via 10.15.1.1

sudo ip -n rtr link set r-c up; sudo ip -n rtr addr add 10.15.1.1/24 dev r-c
sudo ip -n rtr link set r-s up; sudo ip -n rtr addr add 10.15.2.1/24 dev r-s

sudo ip -n srv link set lo up
sudo ip -n srv link set s0 up; sudo ip -n srv addr add 10.15.2.20/24 dev s0
sudo ip -n srv route add default via 10.15.2.1

# Inject Fault 1: IP forwarding left OFF on rtr
sudo ip netns exec rtr sysctl -w net.ipv4.ip_forward=0
# Inject Fault 2: Silent drop of TCP/8080 in rtr forward chain
sudo ip netns exec rtr nft -f - << 'EOF'
table inet fw {
    chain forward {
        type filter hook forward priority filter; policy accept;
        tcp dport 8080 counter drop
    }
}
EOF
# Inject Fault 3: Web server on srv accidentally bound to 127.0.0.1 instead of 0.0.0.0!
sudo ip netns exec srv python3 -m http.server 8080 --bind 127.0.0.1 >/dev/null 2>&1 &
BAD_PID=$!
sleep 0.5

# === SYSTEMATIC TRIAGE & FIX ===
echo "--- Step 1: Test L3 ping (fails due to Fault 1: ip_forward=0) ---"
sudo ip netns exec cli ping -c 1 -W 1 10.15.2.20 || true
sudo ip netns exec rtr sysctl net.ipv4.ip_forward
sudo ip netns exec rtr sysctl -w net.ipv4.ip_forward=1
sudo ip netns exec cli ping -c 2 10.15.2.20   # L3 now works!

echo "--- Step 2: Test HTTP:8080 (hangs due to Fault 2: nftables drop) ---"
sudo ip netns exec cli curl -sI --max-time 2 http://10.15.2.20:8080 || true
# Check nftables drop counters on rtr:
sudo ip netns exec rtr nft list ruleset
# Fix Fault 2: flush blocking table
sudo ip netns exec rtr nft delete table inet fw

echo "--- Step 3: Re-test HTTP:8080 (Connection Refused due to Fault 3: localhost bind!) ---"
sudo ip netns exec cli curl -sI --max-time 2 http://10.15.2.20:8080 || true
# Diagnose on srv with ss -tulnp: notice 127.0.0.1:8080!
sudo ip netns exec srv ss -tulnp
# Fix Fault 3: rebind service to 0.0.0.0:8080
sudo kill "$BAD_PID" 2>/dev/null || true
sudo ip netns exec srv python3 -m http.server 8080 --bind 0.0.0.0 >/dev/null 2>&1 &
GOOD_PID=$!
sleep 0.5

echo "--- Step 4: Final End-to-End Verification ---"
sudo ip netns exec cli curl -sI http://10.15.2.20:8080 | head -n 1

# === CLEANUP ===
sudo kill "$GOOD_PID" 2>/dev/null || true
for ns in cli rtr srv; do sudo ip netns del "$ns"; done
```

**What you verified:** You systematically separated three stacked failures across Layer 3 forwarding (`sysctl`), Netfilter firewalling (`nft` counters), and Layer 4 socket binding (`ss -tulnp`), proving each root cause with hard kernel evidence before fixing it.
