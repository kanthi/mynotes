---
title: Nokia SR Linux Cheat Sheet
---

# Nokia SR Linux Cheat Sheet

Nokia SR Linux is a modern, YANG-native, Linux-based network operating system built around a transactional configuration datastore (`candidate` → `running` → `state`) and native gNMI/JSON-RPC APIs.

## Connecting and Datastore Workflow

```sh
# Interactive SR Linux CLI
docker exec -it clab-LAB-srl1 sr_cli

# Non-interactive single command from host
docker exec clab-LAB-srl1 sr_cli "show version"
docker exec clab-LAB-srl1 sr_cli "show network-instance default protocols bgp neighbor"
```

SR Linux separates **running configuration**, **candidate configuration**, and **operational state**. All edits happen in candidate mode and take effect only when committed.

```text
! Inspect running config and operational state
info from running
info from running interface ethernet-1/1
info flat from running                  ! Show as flat "set" lines
info from state interface ethernet-1/1

! Enter candidate mode, edit, inspect diff, commit
enter candidate
set / interface ethernet-1/1 description "to-spine1"
diff                                    ! Compare candidate vs running
commit now                              ! Commit and exit candidate mode
! Other commit & rollback options:
! commit stay                           ! Commit and stay in candidate mode
! commit confirmed timeout 120          ! Auto-revert after 120s unless confirmed with "commit now"
! discard now                           ! Drop all uncommitted candidate changes
! discard stay                          ! Drop uncommitted changes and stay in candidate mode
```

### Configuration checkpoints and rollback

```text
show system configuration checkpoint
tools system configuration checkpoint create name pre-bgp
tools system configuration revert 1     ! Revert to previous committed revision
save startup                            ! Persist running config to startup config
```

## Interfaces and Subinterfaces

SR Linux separates the physical port (`interface ethernet-1/1`) from logical L2/L3 subinterfaces (`subinterface 0`). Both the parent interface and subinterface must be `admin-state enable`, and the subinterface must be bound to a `network-instance`.

### Routed L3 interface and Loopback (`system0` / `lo0`)

```text
enter candidate
set / interface ethernet-1/1 admin-state enable
set / interface ethernet-1/1 description "to-leaf2"
set / interface ethernet-1/1 subinterface 0 admin-state enable
set / interface ethernet-1/1 subinterface 0 ipv4 admin-state enable
set / interface ethernet-1/1 subinterface 0 ipv4 address 10.1.12.1/30
set / interface ethernet-1/1 subinterface 0 ipv6 admin-state enable
set / interface ethernet-1/1 subinterface 0 ipv6 address 2001:db8:12::1/64

set / interface system0 admin-state enable
set / interface system0 subinterface 0 ipv4 admin-state enable
set / interface system0 subinterface 0 ipv4 address 10.255.0.1/32
commit now
```

### Tagged L2 subinterface and IRB (SVI)

```text
enter candidate
set / interface ethernet-1/2 admin-state enable
set / interface ethernet-1/2 vlan-tagging true
set / interface ethernet-1/2 subinterface 10 type bridged
set / interface ethernet-1/2 subinterface 10 admin-state enable
set / interface ethernet-1/2 subinterface 10 vlan encap single-tagged vlan-id 10

set / interface irb0 admin-state enable
set / interface irb0 subinterface 10 ipv4 admin-state enable
set / interface irb0 subinterface 10 ipv4 address 10.10.10.1/24
commit now
```

### Interface inspection

```text
show interface brief
show interface all
show interface ethernet-1/1 detail
show interface ethernet-1/1 statistics
show system lldp neighbor
show arpnd arp-entries
show arpnd neighbor-entries
```

## Network Instances (`default`, `ip-vrf`, `mac-vrf`)

Every active subinterface belongs to a `network-instance`:
- `default`: the global/underlay routing table
- `ip-vrf`: L3 tenant VRF
- `mac-vrf`: L2 bridge domain / EVPN broadcast domain

```text
enter candidate
! Bind underlay interfaces to default network-instance
set / network-instance default type default
set / network-instance default interface ethernet-1/1.0
set / network-instance default interface system0.0

! Create an L3 ip-vrf
set / network-instance RED type ip-vrf
set / network-instance RED admin-state enable
set / network-instance RED interface irb0.10

! Create an L2 mac-vrf (bridge domain)
set / network-instance VLAN10 type mac-vrf
set / network-instance VLAN10 admin-state enable
set / network-instance VLAN10 interface ethernet-1/2.10
set / network-instance VLAN10 interface irb0.10
commit now
```

Inspect network instances and route tables:

```text
show network-instance summary
show network-instance default route-table ipv4-unicast summary
show network-instance default route-table ipv6-unicast summary
show network-instance RED route-table ipv4-unicast summary
show network-instance VLAN10 bridge-table mac-table all
```

## Static Routes, OSPF, BGP, and EVPN-VXLAN

### Static routes

```text
enter candidate
set / network-instance default next-hop-groups group NH-R2 nexthop 0 ip-address 10.1.12.2
set / network-instance default static-routes route 192.0.2.0/24 next-hop-group NH-R2
commit now
```

### OSPF

```text
enter candidate
set / network-instance default protocols ospf instance main admin-state enable
set / network-instance default protocols ospf instance main version ospf-v2
set / network-instance default protocols ospf instance main router-id 10.255.0.1
set / network-instance default protocols ospf instance main area 0.0.0.0 interface ethernet-1/1.0 interface-type point-to-point
set / network-instance default protocols ospf instance main area 0.0.0.0 interface system0.0 passive true
commit now
```

Inspect OSPF:

```text
show network-instance default protocols ospf instance main neighbor
show network-instance default protocols ospf instance main interface
show network-instance default protocols ospf instance main database
```

### BGP and Routing Policy

> [!IMPORTANT]
> Like FRR, SR Linux enforces RFC 8212 default-deny on eBGP peers: you must attach an `import-policy` and `export-policy` to eBGP peers or peer groups.

```text
enter candidate
! Routing policy (prefix-set + policy)
set / routing-policy prefix-set LOOPBACKS prefix 10.255.0.0/24 mask-length-range 32..32
set / routing-policy policy PERMIT-ALL default-action policy-result accept
set / routing-policy policy EXPORT-LO statement 10 match prefix-set LOOPBACKS
set / routing-policy policy EXPORT-LO statement 10 action policy-result accept

! BGP configuration
set / network-instance default protocols bgp admin-state enable
set / network-instance default protocols bgp autonomous-system 65001
set / network-instance default protocols bgp router-id 10.255.0.1
set / network-instance default protocols bgp afi-safi ipv4-unicast admin-state enable
set / network-instance default protocols bgp group EBGP peer-as 65002
set / network-instance default protocols bgp group EBGP import-policy PERMIT-ALL
set / network-instance default protocols bgp group EBGP export-policy EXPORT-LO
set / network-instance default protocols bgp neighbor 10.1.12.2 peer-group EBGP
commit now
```

Inspect BGP and EVPN:

```text
show network-instance default protocols bgp neighbor
show network-instance default protocols bgp neighbor 10.1.12.2 detail
show network-instance default protocols bgp routes ipv4 summary
show network-instance default protocols bgp routes ipv4 prefix 10.255.0.2/32
show network-instance default protocols bgp neighbor 10.1.12.2 received-routes ipv4
show network-instance default protocols bgp neighbor 10.1.12.2 advertised-routes ipv4
show network-instance default protocols bgp routes evpn summary
show tunnel-interface vxlan0 vxlan-interface
```

## ACLs

```text
enter candidate
set / acl ipv4-filter BLOCK-ICMP entry 10 match protocol icmp
set / acl ipv4-filter BLOCK-ICMP entry 10 action drop
set / acl ipv4-filter BLOCK-ICMP entry 20 action accept
set / acl interface ethernet-1/1.0 input ipv4-filter BLOCK-ICMP
commit now
```

Inspect ACLs:

```text
info acl
show acl ipv4-filter BLOCK-ICMP entry
```

## gNMI and `gnmic` One-Liners

SR Linux exposes its full YANG tree over gNMI (default port `57400`). See the [gNMI lab](../09-lab-workbook/20-gnmi-srlinux.md) for a full walkthrough.

```sh
# Discover supported models / capabilities
gnmic -a clab-LAB-srl1:57400 -u admin -p NokiaSrl1! --insecure capabilities

# Get interface operational state
gnmic -a clab-LAB-srl1:57400 -u admin -p NokiaSrl1! --insecure \
  get --path "/interface[name=ethernet-1/1]/oper-state"

# Stream interface counters every 5 seconds
gnmic -a clab-LAB-srl1:57400 -u admin -p NokiaSrl1! --insecure \
  subscribe --path "/interface[name=ethernet-1/1]/statistics" --sample-interval 5s

# Set an interface description via gNMI
gnmic -a clab-LAB-srl1:57400 -u admin -p NokiaSrl1! --insecure \
  set --update-path "/interface[name=ethernet-1/1]/description" --update-value "managed-by-gnmi"
```

## Linux Bash and Network Namespace Troubleshooting

Each SR Linux `network-instance` maps to a Linux network namespace (`srbase-default`, `srbase-mgmt`, `srbase-<vrf>`). Data-plane ports appear in Linux as `e1-1`, `e1-2`, etc.

```sh
# From host via docker exec, or after typing "bash" in sr_cli:
docker exec clab-LAB-srl1 ip netns list

# Inspect Linux interfaces and routes in the default network-instance
docker exec clab-LAB-srl1 ip netns exec srbase-default ip -br addr
docker exec clab-LAB-srl1 ip netns exec srbase-default ip route show

# Packet capture on ethernet-1/1 inside srbase-default
docker exec clab-LAB-srl1 ip netns exec srbase-default tcpdump -ni e1-1 -vv

# Check listening management services (SSH, gNMI) in srbase-mgmt
docker exec clab-LAB-srl1 ip netns exec srbase-mgmt ss -tulpn
```

> [!TIP]
> You can also run `ping` and `traceroute` directly from `sr_cli` with `ping network-instance default 10.1.12.2` or `traceroute network-instance RED 10.20.1.2`.
