---
title: FRRouting Cheat Sheet
---

# FRRouting Cheat Sheet

FRRouting (FRR) is a suite of modular routing daemons running on Linux. Linux owns the kernel interfaces, IP addresses, VRFs, and forwarding table (FIB), while FRR daemons compute routes and push them into the kernel through `zebra`. Use Linux `ip` tools to verify the data plane and `vtysh` to manage the control plane.

## Architecture and `/etc/frr/daemons`

Each routing protocol runs as its own daemon. A daemon's CLI commands are only active if that daemon is enabled in `/etc/frr/daemons` before FRR starts.

```text
/etc/frr/daemons      # Enables/disables protocol daemons (bgpd=yes, ospfd=yes, ...)
/etc/frr/frr.conf     # Unified integrated configuration (recommended)
/etc/frr/vtysh.conf   # Enables "service integrated-vtysh-config"
```

Common daemons to enable in `/etc/frr/daemons`:

| Daemon | Purpose |
|---|---|
| `zebra` | Always required; kernel RIB/FIB synchronization, interfaces, VRFs |
| `staticd` | Static IPv4/IPv6 routes |
| `bgpd` | BGP-4, MP-BGP (IPv6, VPN, EVPN) |
| `ospfd` / `ospf6d` | OSPFv2 (IPv4) and OSPFv3 (IPv6) |
| `isisd` | IS-IS (IPv4/IPv6, Segment Routing) |
| `bfdd` | Bidirectional Forwarding Detection |
| `mgmtd` | Management daemon (FRR 9+) |

Check which daemons are running from Linux or inside `vtysh`:

```sh
docker exec clab-LAB-r1 vtysh -c 'show daemons'
docker exec clab-LAB-r1 ps aux | grep -E 'zebra|bgpd|ospfd|isisd|bfdd'
```

## Connecting and `vtysh` One-Liners

`vtysh` aggregates all active daemons into a single Cisco-like CLI. You can open an interactive shell or pass one or more `-c` flags from bash or `docker exec`.

```sh
# Interactive session
docker exec -it clab-LAB-r1 vtysh

# Single show command (great for scripts and lab validation)
docker exec clab-LAB-r1 vtysh -c 'show bgp summary'

# Multiple commands in sequence (configuration one-liner)
docker exec clab-LAB-r1 vtysh \
  -c 'configure terminal' \
  -c 'ip route 192.0.2.0/24 10.1.12.2' \
  -c 'end' \
  -c 'show ip route 192.0.2.0/24'
```

Inside `vtysh`, you can also inspect specific config sections without dumping the entire file:

```text
show running-config
show running-config bgpd
show running-config ospfd
show running-config zebra
show version
```

## Interfaces and VRFs

Linux creates the VRF device and binds interfaces to it; FRR detects the VRF automatically through `zebra` and exposes it as `vrf <NAME>`.

### Linux VRF setup (host/container shell)

```sh
ip link add RED type vrf table 100
ip link set RED up
ip link set eth2 master RED
ip addr add 10.10.1.1/24 dev eth2
ip link set eth2 up
```

### FRR interface and VRF inspection (`vtysh`)

```text
show interface brief
show interface eth1
show vrf
show vrf vni
show ip route
show ip route vrf RED
show ipv6 route vrf RED
```

### Configure an interface address inside FRR

```text
configure terminal
interface eth1
 description to-r2
 ip address 10.1.12.1/30
 ipv6 address 2001:db8:12::1/64
 no shutdown
end
```

## Static Routes

Static routes are managed by `staticd` and installed into the kernel RIB by `zebra` only when the next hop is reachable.

```text
configure terminal
! Standard IPv4 and IPv6 static routes
ip route 192.0.2.0/24 10.1.12.2
ipv6 route 2001:db8:200::/48 2001:db8:12::2

! Floating static (higher administrative distance, e.g., 200)
ip route 192.0.2.0/24 10.1.13.2 200

! Null / blackhole route (for summary advertisement or drop)
ip route 198.51.100.0/24 blackhole

! VRF static route
vrf RED
 ip route 10.99.0.0/24 10.10.1.2
exit-vrf
end
```

Verify static routes and kernel installation:

```text
show ip route static
show ip route 192.0.2.0/24
show ip fib 192.0.2.0/24
```

## OSPF and OSPFv3

### Minimal OSPFv2 configuration

```text
configure terminal
interface lo
 ip address 10.255.0.1/32
 ip ospf area 0
!
interface eth1
 ip ospf area 0
 ip ospf network point-to-point
!
router ospf
 ospf router-id 10.255.0.1
 passive-interface lo
end
```

### Minimal OSPFv3 (IPv6) configuration

```text
configure terminal
interface eth1
 ipv6 ospf6 area 0.0.0.0
 ipv6 ospf6 network point-to-point
!
router ospf6
 ospf6 router-id 10.255.0.1
end
```

### OSPF inspection commands

```text
show ip ospf
show ip ospf neighbor
show ip ospf neighbor detail
show ip ospf interface brief
show ip ospf database
show ip ospf database router
show ip ospf route
show ipv6 ospf6 neighbor
show ipv6 ospf6 route
```

## IS-IS

### Minimal IS-IS configuration

```text
configure terminal
interface lo
 ip router isis CORE
 ipv6 router isis CORE
 isis passive
!
interface eth1
 ip router isis CORE
 ipv6 router isis CORE
 isis network point-to-point
!
router isis CORE
 net 49.0001.0102.5500.0001.00
 is-type level-2-only
 metric-style wide
end
```

### IS-IS inspection commands

```text
show isis summary
show isis neighbor
show isis neighbor detail
show isis interface
show isis database
show isis database detail
show isis route
```

## BGP and EVPN

> [!IMPORTANT]
> FRR defaults to RFC 8212 (`bgp ebgp-requires-policy`): an eBGP peer will **neither send nor accept prefixes** until you apply an inbound and outbound route-map (or explicitly configure `no bgp ebgp-requires-policy` in a lab).

### eBGP with prefix-lists, route-maps, and communities

```text
configure terminal
ip prefix-list PL-OUT seq 10 permit 10.255.0.1/32
ip prefix-list PL-OUT seq 20 permit 192.0.2.0/24 le 24
!
bgp community-list standard CL-LOCAL seq 10 permit 65001:100
!
route-map RM-OUT permit 10
 match ip address prefix-list PL-OUT
 set community 65001:100 additive
!
route-map RM-IN permit 10
 match community CL-LOCAL
 set local-preference 200
route-map RM-IN permit 20
!
router bgp 65001
 bgp router-id 10.255.0.1
 neighbor 10.1.12.2 remote-as 65002
 neighbor 10.1.12.2 description r2
 !
 address-family ipv4 unicast
  network 10.255.0.1/32
  neighbor 10.1.12.2 soft-reconfiguration inbound
  neighbor 10.1.12.2 route-map RM-IN in
  neighbor 10.1.12.2 route-map RM-OUT out
 exit-address-family
end
```

### BGP EVPN (L2VPN EVPN)

```text
configure terminal
router bgp 65001
 neighbor 10.255.0.2 remote-as 65001
 neighbor 10.255.0.2 update-source lo
 !
 address-family l2vpn evpn
  neighbor 10.255.0.2 activate
  advertise-all-vni
 exit-address-family
end
```

### Essential BGP and EVPN inspection commands

```text
show bgp summary
show bgp ipv4 unicast summary
show bgp ipv6 unicast summary
show bgp l2vpn evpn summary
show bgp neighbors 10.1.12.2
show bgp ipv4 unicast
show bgp ipv4 unicast 203.0.113.0/24
show bgp ipv4 unicast neighbors 10.1.12.2 received-routes
show bgp ipv4 unicast neighbors 10.1.12.2 routes
show bgp ipv4 unicast neighbors 10.1.12.2 advertised-routes
show bgp ipv4 unicast community 65001:100
show bgp l2vpn evpn
show bgp l2vpn evpn route type macip
show evpn vni
show evpn mac vni all
show evpn arp-cache vni all
```

### Soft reconfiguration and route refresh

```text
clear bgp 10.1.12.2 soft in
clear bgp 10.1.12.2 soft out
clear bgp * soft
clear bgp ipv4 unicast 10.1.12.2
```

## BFD

Enable `bfdd=yes` in `/etc/frr/daemons`, define a profile, and attach BFD to BGP or OSPF neighbors.

```text
configure terminal
bfd
 profile FAST
  detect-multiplier 3
  transmit-interval 100
  receive-interval 100
 !
!
router bgp 65001
 neighbor 10.1.12.2 bfd profile FAST
!
interface eth1
 ip ospf bfd
 ip ospf bfd profile FAST
end
```

Verify BFD sessions:

```text
show bfd peers
show bfd peers brief
show bfd peers counters
```

## Debugging and Troubleshooting

Always cross-check FRR's RIB (`show ip route`) against Linux's kernel FIB (`ip route get`). If a route is in BGP (`show bgp ipv4 unicast`) but missing from `show ip route`, check next-hop reachability (`show bgp ipv4 unicast <prefix>`).

```text
! Live debugging inside vtysh (turn off with "no debug ..." or "undebug all")
debug bgp neighbor-events
debug bgp updates
debug ospf event
debug zebra kernel
terminal monitor

! Disable all debugging
undebug all
```

From the Linux shell of the FRR container:

```sh
ip -br addr
ip route get 192.0.2.1
ip neigh show
sudo tcpdump -ni eth1 'tcp port 179 or proto ospf'
```

## Configuration Persistence in Containerlab

`write memory` writes the running configuration to `/etc/frr/frr.conf` inside the container. If you destroy and redeploy the lab, container-local changes are lost unless `/etc/frr/frr.conf` is bind-mounted from your host directory.

```sh
# Save running config inside the container
docker exec clab-LAB-r1 vtysh -c 'write memory'

# Export running config to a host file before destroying a lab
docker exec clab-LAB-r1 vtysh -c 'show running-config' > configs/r1-frr.conf
```

> [!TIP]
> In Containerlab topologies, bind-mount both `daemons` and `frr.conf` into `/etc/frr/` so your lab is reproducible from a clean `containerlab deploy`. See [Containerlab, netlab, and Linux Cheat Sheet](05-lab-tools.md) for `containerlab save` and mount tips.
