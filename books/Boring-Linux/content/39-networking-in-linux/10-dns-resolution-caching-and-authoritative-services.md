---
title: 'Chapter 10: DNS: Resolution, Caching, and Authoritative Services'
---

# Chapter 10: DNS: Resolution, Caching, and Authoritative Services

"It's always DNS." On Linux, name resolution is a multi-stage chain spanning C library Name Service Switch (NSS), local stub listeners (`systemd-resolved` at `127.0.0.53`), split-DNS VPN routing domains, recursive DNSSEC-validating caches (**Unbound**), and authoritative zone servers (**BIND 9**). Mastering every link in that chain eliminates hours of blind troubleshooting.

## How Linux Resolves a Hostname (NSS → Stub → Upstream)

When an application calls `getaddrinfo("api.corp.internal", ...)`:

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                    LINUX HOSTNAME RESOLUTION PATH                       │
│                                                                         │
│   Application (curl / ssh / Python / Go)                                │
│         │  getaddrinfo()                                                │
│         ▼                                                               │
│   /etc/nsswitch.conf ──► hosts: files resolve [!UNAVAIL=return] dns     │
│         │                                                               │
│         ├── 1. files   ──► /etc/hosts                                   │
│         ├── 2. resolve ──► D-Bus/varlink directly to systemd-resolved   │
│         └── 3. dns     ──► /etc/resolv.conf (127.0.0.53 or upstream IP) │
│                                │                                        │
│                                ▼                                        │
│                        systemd-resolved                                 │
│                   (Per-link Split-DNS + Cache + DoT + DNSSEC)           │
│                         ╱                   ╲                           │
│           ~corp.internal                     . (default)                │
│                 ▼                                 ▼                     │
│     [Corp DNS via wg0: 10.10.0.53]      [Unbound / 1.1.1.1 DoT]         │
└─────────────────────────────────────────────────────────────────────────┘
```

> [!NOTE]
> **Static Go binaries & `dig` bypass NSS!** Tools like `dig`, `drill`, and Go programs compiled with `CGO_ENABLED=0` read `/etc/resolv.conf` directly and skip `/etc/nsswitch.conf` (`resolve`, `mdns`, `/etc/hosts` order may differ). Always test with **`getent ahosts <name>`** to see what normal Linux applications see, and **`dig`** to test raw DNS protocol behavior.

## Mastering `systemd-resolved` and Split-DNS

Ubuntu 26.04 LTS, Fedora, and modern Debian servers use **`systemd-resolved`** as the local caching stub resolver (`127.0.0.53:53` and full proxy mode at `127.0.0.54:53`).

### Understanding `/etc/resolv.conf` Modes

| `/etc/resolv.conf` Target | Mode | Behavior |
|---|---|---|
| Symlink → `/run/systemd/resolve/stub-resolv.conf` | **Stub (default)** | Points to `127.0.0.53`; enables per-interface Split-DNS, local cache, DoT, DNSSEC |
| Symlink → `/run/systemd/resolve/resolv.conf` | **Uplink** | Lists upstream servers directly (useful when containers mount host `/etc/resolv.conf`) |
| Static regular file | **Foreign** | `systemd-resolved` consumes it as global upstream instead of managing it |

### Essential `resolvectl` Operations

```bash
# Inspect global and per-link DNS servers, routing domains (~domain), DNSSEC & DoT status
resolvectl status

# Test resolution through systemd-resolved (shows which link & cache hit/miss answered!)
resolvectl query api.corp.internal

# Inspect cache hit ratio and DNSSEC validation counters
resolvectl statistics

# Flush local DNS cache immediately
sudo resolvectl flush-caches

# Configure Split-DNS dynamically for a WireGuard VPN link (wg0):
# The leading '~' makes corp.internal a ROUTING-ONLY domain (not appended as a search suffix)
sudo resolvectl dns wg0 10.10.0.53
sudo resolvectl domain wg0 '~corp.internal' '~10.in-addr.arpa'
```

### Hardening `/etc/systemd/resolved.conf.d/99-hardened.conf`

```ini
[Resolve]
DNS=1.1.1.1#cloudflare-dns.com 9.9.9.9#dns.quad9.net
FallbackDNS=8.8.8.8#dns.google
DNSOverTLS=yes
DNSSEC=yes
Cache=yes
LLMNR=no
MulticastDNS=no
```

> [!TIP]
> Disabling `LLMNR=no` and `MulticastDNS=no` on servers eliminates Responder-style LAN credential poisoning attacks.

## Recursive Caching DNS with Unbound

When running your own recursive resolver for an office, lab, or Kubernetes cluster, **Unbound** is fast, minimal, memory-safe, and validates DNSSEC out of the box — either resolving directly from the Internet Root Hints (no third-party DNS dependency) or forwarding over encrypted **DNS-over-TLS (DoT)**.

```yaml
# /etc/unbound/unbound.conf.d/10-recursive.conf
server:
    interface: 0.0.0.0
    interface: ::0
    port: 53
    do-ip4: yes
    do-ip6: yes
    do-udp: yes
    do-tcp: yes

    # Access Control (NEVER run an open Internet resolver!)
    access-control: 127.0.0.0/8 allow
    access-control: 10.0.0.0/8 allow
    access-control: 192.168.0.0/16 allow
    access-control: 0.0.0.0/0 refuse

    # Privacy & DNSSEC hardening
    qname-minimisation: yes
    harden-glue: yes
    harden-dnssec-stripped: yes
    hide-identity: yes
    hide-version: yes
    prefetch: yes
    cache-min-ttl: 60
    num-threads: 4

    # Internal private zone overrides (stub-zone / local-data)
    private-domain: "corp.internal"
    local-zone: "corp.internal." static
    local-data: "gateway.corp.internal. IN A 10.10.10.1"
    local-data-ptr: "10.10.10.1 gateway.corp.internal"

# Forward corp.internal to internal authoritative BIND 9 server
stub-zone:
    name: "prod.internal"
    stub-addr: 10.10.10.54
```

Validate and inspect Unbound:

```bash
sudo unbound-checkconf
sudo systemctl enable --now unbound
sudo unbound-control stats_noreset | grep -E 'total.num.queries|total.num.cachehits'
```

## Authoritative DNS with BIND 9 (`named`)

**BIND 9** remains the gold standard for authoritative internal/external DNS zones, primary/secondary zone transfers (`AXFR`/`IXFR`) authenticated with TSIG keys, Dynamic DNS updates, and Split-Horizon Views.

### Primary Zone Configuration (`/etc/bind/named.conf.local`)

```text
key "axfr-key" {
    algorithm hmac-sha256;
    secret "BASE64_GENERATED_BY_TSIG_KEYGEN==";
};

zone "lab.internal" {
    type primary;
    file "/etc/bind/zones/db.lab.internal";
    allow-transfer { key "axfr-key"; };
    also-notify { 10.10.10.55; };
};

zone "10.10.10.in-addr.arpa" {
    type primary;
    file "/etc/bind/zones/db.10.10.10";
};
```

### Forward Zone File (`/etc/bind/zones/db.lab.internal`)

```text
$TTL    3600
@       IN      SOA     ns1.lab.internal. hostmaster.lab.internal. (
                        2026111901      ; Serial (YYYYMMDDNN — MUST increment on every edit!)
                        3600            ; Refresh
                        900             ; Retry
                        604800          ; Expire
                        300 )           ; Negative Cache TTL

; Authoritative Nameservers
@       IN      NS      ns1.lab.internal.
ns1     IN      A       10.10.10.54

; Host A / AAAA / CNAME / SRV records
rtr01   IN      A       10.10.10.1
rtr01   IN      AAAA    fd00:10::1
web01   IN      A       10.10.10.20
app     IN      CNAME   web01.lab.internal.
_https._tcp IN  SRV     10 5 443 web01.lab.internal.
```

Always validate both config and zone files before reloading BIND:

```bash
sudo named-checkconf
sudo named-checkzone lab.internal /etc/bind/zones/db.lab.internal
sudo rndc reload lab.internal
```

## Diagnostic Mastery: `dig`, `delv`, and `getent`

| Command | What It Checks |
|---|---|
| `getent ahosts web01.lab.internal` | Exact NSS resolution path used by system apps (`/etc/hosts` + `systemd-resolved`) |
| `dig @10.10.10.54 web01.lab.internal A +noall +answer +stats` | Raw DNS answer + latency from a specific server |
| `dig +trace example.com` | Full iterative delegation walk from `.` Root → TLD → Authoritative NS |
| `dig +dnssec +multi cloudflare.com A` | Shows `ad` (Authenticated Data) flag and `RRSIG` cryptographic signatures |
| `delv @1.1.1.1 cloudflare.com` | Performs independent local DNSSEC chain-of-trust validation |
| `dig -x 10.10.10.1` | Reverse PTR lookup (`1.10.10.10.in-addr.arpa`) |
| `dig @10.10.10.54 lab.internal AXFR` | Tests zone transfer permissions (should return `Transfer failed` from untrusted IPs!) |

Understanding critical DNS response codes (`status:` in `dig`):
- **`NOERROR`**: Query succeeded (even if ANSWER section is empty, e.g., asking for `AAAA` when only `A` exists).
- **`NXDOMAIN`**: Domain does not exist (cached for SOA Negative TTL).
- **`SERVFAIL`**: Upstream unreachable, zone broken, or **DNSSEC signature validation failed**! Test with `dig +cd` (Checking Disabled) to confirm whether a `SERVFAIL` is a DNSSEC break.
- **`REFUSED`**: Server ACL (`access-control` / `allow-query`) rejected the client IP.

## Hands-On Lab 10: Authoritative BIND/dnsmasq + Caching Unbound Forwarding Lab

Build a two-tier enterprise DNS hierarchy inside namespaces: `client` queries `unbound-cache` (`10.100.0.53`), which caches responses and forwards the internal domain `corp.lab` to an authoritative `dnsmasq` server (`10.100.0.54`).

```bash
# 1. Create namespaces connected via a switch bridge
for ns in client cache auth sw; do sudo ip netns add "$ns"; done

sudo ip link add c0 type veth peer name sw-c
sudo ip link add u0 type veth peer name sw-u
sudo ip link add a0 type veth peer name sw-a
sudo ip link set c0 netns client; sudo ip link set sw-c netns sw
sudo ip link set u0 netns cache;  sudo ip link set sw-u netns sw
sudo ip link set a0 netns auth;   sudo ip link set sw-a netns sw

sudo ip -n sw link add br0 type bridge
sudo ip -n sw link set br0 up
for p in sw-c sw-u sw-a; do sudo ip -n sw link set "$p" master br0 up; done

sudo ip -n client link set lo up; sudo ip -n client link set c0 up; sudo ip -n client addr add 10.100.0.10/24 dev c0
sudo ip -n cache  link set lo up; sudo ip -n cache  link set u0 up; sudo ip -n cache  addr add 10.100.0.53/24 dev u0
sudo ip -n auth   link set lo up; sudo ip -n auth   link set a0 up; sudo ip -n auth   addr add 10.100.0.54/24 dev a0

# 2. Start authoritative DNS server for corp.lab inside namespace 'auth'
mkdir -p /tmp/lab10-dns
sudo ip netns exec auth dnsmasq \
  --interface=a0 --bind-interfaces --no-resolv --no-hosts \
  --address=/api.corp.lab/10.100.0.200 \
  --address=/db.corp.lab/10.100.0.201 \
  --host-record=gateway.corp.lab,10.100.0.1 \
  --pid-file=/tmp/lab10-dns/auth.pid

# 3. Start Unbound caching resolver inside namespace 'cache', stub-forwarding corp.lab -> 10.100.0.54
cat << 'EOF' > /tmp/lab10-dns/unbound.conf
server:
    interface: 10.100.0.53
    port: 53
    do-ip4: yes
    access-control: 10.100.0.0/24 allow
    do-not-query-localhost: no
    domain-insecure: "corp.lab"
    private-domain: "corp.lab"
    logfile: ""
    use-syslog: no
    pidfile: "/tmp/lab10-dns/unbound.pid"
forward-zone:
    name: "corp.lab."
    forward-addr: 10.100.0.54
EOF
sudo ip netns exec cache unbound -c /tmp/lab10-dns/unbound.conf

# 4. Query from 'client' against the Unbound cache (10.100.0.53)
#    First query hits auth (non-zero query time); second query is served from Unbound cache (0 msec, decreasing TTL)!
sudo ip netns exec client dig @10.100.0.53 api.corp.lab A +noall +answer +stats
sleep 1
sudo ip netns exec client dig @10.100.0.53 api.corp.lab A +noall +answer +stats
# Reverse PTR lookup for host-record
sudo ip netns exec client dig @10.100.0.53 -x 10.100.0.1 +short

# 5. Cleanup
sudo kill "$(cat /tmp/lab10-dns/auth.pid)" 2>/dev/null || true
sudo kill "$(cat /tmp/lab10-dns/unbound.pid)" 2>/dev/null || true
rm -rf /tmp/lab10-dns
for ns in client cache auth sw; do sudo ip netns del "$ns"; done
```

**What you verified:** You built a two-tier enterprise DNS pipeline (authoritative zone server behind a caching recursive forwarder) and observed cache hit timing and TTL countdown directly in `dig`.
