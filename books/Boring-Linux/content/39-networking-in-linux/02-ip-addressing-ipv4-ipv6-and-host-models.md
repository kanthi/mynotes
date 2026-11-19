---
title: 'Chapter 2: IP Addressing, IPv4/IPv6, and Host Models'
---

# Chapter 2: IP Addressing, IPv4/IPv6, and Host Models

On a traditional router, an IP address feels glued to a physical port. In Linux, IP addressing is far more flexible — and carries subtle behaviors around primary vs secondary addresses, address scopes, IPv6 Stateless Address Autoconfiguration (SLAAC), source-address selection, and the **Weak Host Model** that routinely surprise engineers building multi-homed servers or routers.

## IPv4 and IPv6 Addresses in the Linux Kernel

In Linux, a single `net_device` can hold an arbitrary number of IPv4 (`in_ifaddr`) and IPv6 (`inet6_ifaddr`) addresses simultaneously, from completely different subnets, without requiring legacy alias interfaces like `eth0:0`.

```bash
# Add primary and additional IPv4 addresses on one interface
sudo ip addr add 192.0.2.10/24 dev enp1s0
sudo ip addr add 192.0.2.11/24 dev enp1s0
sudo ip addr add 198.51.100.5/24 dev enp1s0

# Add a /32 service VIP or loopback IP
sudo ip addr add 203.0.113.50/32 dev lo

# Add an IPv6 Global Unicast Address
sudo ip -6 addr add 2001:db8:1::10/64 dev enp1s0

# Remove a specific address (always include prefix length!)
sudo ip addr del 192.0.2.11/24 dev enp1s0

# Flush all IPv4 addresses on an interface (NEVER run over SSH on your active interface!)
sudo ip -4 addr flush dev enp1s0
```

### Primary vs Secondary IPv4 Addresses (`promote_secondaries`)

When you add multiple IPv4 addresses from the **same subnet** to an interface, Linux marks the first one as **primary** and subsequent ones as `secondary`:

```text
inet 192.0.2.10/24 scope global enp1s0
   valid_lft forever preferred_lft forever
inet 192.0.2.11/24 scope global secondary enp1s0
   valid_lft forever preferred_lft forever
```

> [!WARNING]
> **Critical trap:** By default on some kernels/configurations (`net.ipv4.conf.all.promote_secondaries = 0`), deleting the **primary** IPv4 address (`192.0.2.10/24`) automatically deletes **all secondary addresses** in that same subnet (`192.0.2.11/24`)! Always enable promotion on multi-IP hosts and Keepalived/VRRP nodes:
> ```bash
> sudo sysctl -w net.ipv4.conf.all.promote_secondaries=1
> sudo sysctl -w net.ipv4.conf.default.promote_secondaries=1
> ```

### Address Scopes and Flags

Every Linux IP address carries a **scope** that controls source-address selection (RFC 6724) and routing validity:

| Scope | Numeric Value | Meaning |
|---|---|---|
| `global` | `0` | Valid everywhere (public IPs, RFC 1918 routed IPs, IPv6 GUAs) |
| `site` | `200` | Site-local (deprecated IPv6 `fec0::/10`, rarely used) |
| `link` | `253` | Valid only on the directly attached Layer 2 segment (`169.254.0.0/16`, `fe80::/10`) |
| `host` | `254` | Valid only inside this host (`127.0.0.1/8`, `::1/128`, or host-scoped VIPs) |

Special flags you will see in `ip addr show`:

- `noprefixroute`: Adds the address **without** automatically installing the connected subnet route (essential when a routing daemon or WireGuard manages routes explicitly).
- `nodad`: Skips IPv6 Duplicate Address Detection (useful for fast failover anycast VIPs or point-to-point links).
- `tentative`: IPv6 address is currently running DAD and cannot yet send/receive traffic.
- `dadfailed`: Another host on the link already owns this IPv6 address!
- `deprecated`: Valid for existing connections (`valid_lft > 0`), but Linux will not choose it as the source IP for new outbound sockets (`preferred_lft = 0`).

## IPv6 Addressing Lifecycle: Link-Local, SLAAC, DAD, and Privacy

Unlike IPv4, IPv6 is mandatory for core protocols like Neighbor Discovery (NDP) and Router Advertisements (RAs):

1. **Link-Local Auto-Generation (`fe80::/64`):** The instant an interface is set `UP`, the kernel generates a link-local address (via EUI-64 or RFC 7217 stable-privacy hash).
2. **Duplicate Address Detection (DAD):** Before using any new unicast IPv6 address, Linux joins the solicited-node multicast group (`ff02::1:ffXX:XXXX`) and transmits a Neighbor Solicitation from `::`. If no Neighbor Advertisement arrives within `dad_transmits` attempts, the address leaves `tentative` state and becomes active.
3. **Router Solicitation (RS) & Router Advertisement (RA):** If `accept_ra = 1` (default when forwarding is off), the host sends RS (`ff02::2`) and configures prefixes advertised with the `A` (Autonomous) flag via SLAAC, plus a default route via the router's `fe80::` link-local address.
4. **Temporary Privacy Addresses (RFC 4941 / 8981):** Controlled by `net.ipv6.conf.<iface>.use_tempaddr`:
   - `0`: Disabled (recommended for servers so outbound firewall allowlists stay stable).
   - `1`: Generate temporary addresses, but prefer stable address for outbound connections.
   - `2`: Generate and prefer temporary addresses for outbound connections (default on desktops).

> [!IMPORTANT]
> **Router gotcha:** Enabling `net.ipv6.conf.all.forwarding = 1` automatically disables RA acceptance (`accept_ra = 1` stops working unless explicitly set to `2`!). If your Linux router receives its WAN IPv6 default route via ISP Router Advertisements, you **must** set:
> ```bash
> sudo sysctl -w net.ipv6.conf.wan0.accept_ra=2
> ```

### Key IPv6 Addressing Sysctls

```ini
# /etc/sysctl.d/20-ipv6-addressing.conf
# Disable temporary privacy addresses on servers
net.ipv6.conf.all.use_tempaddr = 0
net.ipv6.conf.default.use_tempaddr = 0

# Allow binding to non-local IPs (essential for HAProxy / Keepalived / Anycast)
net.ipv4.ip_nonlocal_bind = 1
net.ipv6.ip_nonlocal_bind = 1

# Keep IPv6 addresses when link momentarily flaps down (kernel 4.x+)
net.ipv6.conf.all.keep_addr_on_down = 1
```

## Source Address Selection in Linux

When a process opens an outbound connection without calling `bind()` to a specific IP, how does Linux pick the source IP?

- **For IPv4:**
  1. If the matched route has an explicit `src <ip>` hint (`ip route show`), Linux uses it.
  2. Otherwise, Linux picks the primary IPv4 address on the egress interface whose scope matches the route scope.
  3. You can steer source IP selection cleanly per route:
     ```bash
     sudo ip route replace default via 192.0.2.1 dev enp1s0 src 192.0.2.11
     ```
- **For IPv6 (RFC 6724):** Linux evaluates a deterministic 8-rule table (same address, matching scope, avoid deprecated, prefer home address, prefer outgoing interface, matching label in `ip addrlabel`, prefer public/temp per `use_tempaddr`, longest matching prefix). Inspect the kernel's IPv6 address selection policy table with:
  ```bash
  ip addrlabel show
  ```

## Weak Host Model vs Strong Host Model (and ARP Flux)

This is one of the single most important behavioral differences between Linux and network appliances.

- **Strong Host Model:** An interface only accepts packets whose destination IP is assigned to *that specific interface*, and only answers ARP for IPs on *that specific interface*.
- **Weak Host Model (Linux default for IPv4):** All local IP addresses live in a shared `local` routing table (`ip route show table local`). If a server has `eth0 = 10.0.0.10/24` and `eth1 = 10.0.0.11/24` on the same L2 VLAN (or reachable broadcast domain):
  - An ARP Who-Has for `10.0.0.10` arriving on `eth1` receives a reply from `eth1` containing **`eth1`'s MAC address**!
  - This phenomenon is called **ARP Flux**, and it causes traffic meant for `eth0` to bounce unpredictably onto `eth1`.

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                    ARP FLUX UNDER DEFAULT WEAK HOST                     │
│                                                                         │
│   Switch Broadcast: "Who has 10.0.0.10?"                                │
│         ├──► eth0 (10.0.0.10, MAC aa:aa) ──► Replies: "10.0.0.10 is aa" │
│         └──► eth1 (10.0.0.11, MAC bb:bb) ──► Replies: "10.0.0.10 is bb" │
│                                                                         │
│   Result: Upstream switch/hosts randomly flip MAC entries between       │
│           eth0 and eth1!                                                │
└─────────────────────────────────────────────────────────────────────────┘
```

### Taming ARP Flux with `arp_ignore` and `arp_announce`

Configure these sysctls on any multi-homed Linux host, load balancer (LVS/DSR), or router:

```ini
# /etc/sysctl.d/25-arp-behavior.conf

# arp_ignore = 1: Reply to ARP requests ONLY if the target IP is configured
# on the incoming interface itself.
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.default.arp_ignore = 1

# arp_announce = 2: Always use the best local address assigned to the outgoing
# interface as the ARP sender IP (ignore IP source field from upper layer).
net.ipv4.conf.all.arp_announce = 2
net.ipv4.conf.default.arp_announce = 2

# arp_filter = 1: Only answer ARP if kernel route back to requester exits same interface.
net.ipv4.conf.all.arp_filter = 1
```

| `arp_ignore` Value | Behavior |
|---|---|
| `0` (default) | Reply for any local IP configured on *any* interface |
| `1` | Reply only if target IP is configured on the *incoming* interface |
| `2` | Same as `1`, plus sender IP must belong to the same subnet as target IP |
| `3` | Do not reply for `host`-scoped addresses |
| `8` | Never reply to ARP requests (used on `lo` for LVS Direct Server Return VIPs) |

## Hands-On Lab 2: Observing ARP Flux, Fixing It, and Testing Source Selection

Build a dual-homed server (`srv`) connected via a Linux bridge (`sw`) to a client (`cli`), witness ARP flux live, and eliminate it with `arp_ignore`/`arp_announce`.

```bash
# 1. Create namespaces: client (cli), switch (sw), dual-homed server (srv)
sudo ip netns add cli
sudo ip netns add sw
sudo ip netns add srv

# 2. Wire cli <-> sw and dual links srv(eth0, eth1) <-> sw
sudo ip link add cli0 type veth peer name sw-cli
sudo ip link add srv0 type veth peer name sw-srv0
sudo ip link add srv1 type veth peer name sw-srv1

sudo ip link set cli0 netns cli
sudo ip link set sw-cli netns sw
sudo ip link set sw-srv0 netns sw
sudo ip link set sw-srv1 netns sw
sudo ip link set srv0 netns srv
sudo ip link set srv1 netns srv

# 3. Configure the switch bridge in namespace sw
sudo ip -n sw link add br0 type bridge
sudo ip -n sw link set br0 up
for p in sw-cli sw-srv0 sw-srv1; do
  sudo ip -n sw link set "$p" master br0 up
done

# 4. Configure IPs on cli and dual-homed srv (same subnet 10.20.0.0/24)
sudo ip -n cli link set cli0 up
sudo ip -n cli addr add 10.20.0.1/24 dev cli0

sudo ip -n srv link set srv0 address 02:00:00:00:00:10 up
sudo ip -n srv link set srv1 address 02:00:00:00:00:11 up
sudo ip -n srv addr add 10.20.0.10/24 dev srv0
sudo ip -n srv addr add 10.20.0.11/24 dev srv1

# 5. Capture ARP replies on cli while pinging 10.20.0.11
#    With default weak host, BOTH srv0 (..:10) and srv1 (..:11) reply for 10.20.0.11!
sudo ip netns exec cli timeout 3 tcpdump -l -nn -e -i cli0 arp &
sleep 0.5
sudo ip netns exec cli ping -c 1 10.20.0.11
wait

# 6. Enable strict ARP ownership on srv
sudo ip netns exec srv sysctl -w net.ipv4.conf.all.arp_ignore=1
sudo ip netns exec srv sysctl -w net.ipv4.conf.all.arp_announce=2

# 7. Flush ARP cache on cli and re-test — now ONLY 02:00:00:00:00:11 replies for 10.20.0.11!
sudo ip -n cli neigh flush all
sudo ip netns exec cli timeout 3 tcpdump -l -nn -e -i cli0 arp &
sleep 0.5
sudo ip netns exec cli ping -c 1 10.20.0.11
wait
sudo ip -n cli neigh show

# 8. Test secondary IP promotion on srv
sudo ip netns exec srv sysctl -w net.ipv4.conf.all.promote_secondaries=1
sudo ip -n srv addr add 10.20.0.12/24 dev srv0
sudo ip -n srv addr del 10.20.0.10/24 dev srv0
sudo ip -n srv -br addr show dev srv0   # 10.20.0.12/24 survived and was promoted!

# 9. Cleanup
sudo ip netns del cli
sudo ip netns del sw
sudo ip netns del srv
```

**Key takeaway:** Understanding address scopes, `promote_secondaries`, `accept_ra=2`, and `arp_ignore`/`arp_announce` prevents the most insidious intermittent failures on multi-NIC Linux servers and routers.
