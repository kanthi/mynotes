---
title: Cisco IOS-XE, NX-OS, and Juniper Junos Quick Reference
---

# Cisco IOS-XE, NX-OS, and Juniper Junos Quick Reference

In production networks and mixed-vendor labs, you frequently translate between **Cisco IOS-XE**, **Cisco NX-OS**, and **Juniper Junos** alongside FRR, EOS, and SR Linux. This page collects everyday show, configuration, commit, rollback, and troubleshooting commands side by side.

## CLI Modes, Commit, Rollback, and Saving

### Cisco IOS-XE and NX-OS

IOS-XE and NX-OS apply configuration lines immediately when you press Enter in `configure terminal`. Changes persist across reloads only after saving to `startup-config`.

```text
! Inspect running config and diffs
show running-config
show running-config | section router bgp
show archive config differences nvram:startup-config system:running-config   ! IOS-XE
show diff rollback-patch running-config startup-config                      ! NX-OS

! Safe timed rollback on IOS-XE (reverts in 5 min unless confirmed)
configure terminal revert timer 5
! ... make changes ...
configure confirm

! Rollback to a checkpoint (NX-OS)
checkpoint PRE-CHANGE
rollback running-config checkpoint PRE-CHANGE

! Save running config to startup config
copy running-config startup-config    ! or: write memory (IOS-XE)
```

### Juniper Junos

Junos always edits a **candidate** configuration and applies changes atomically on `commit`. Every commit automatically saves the previous configuration as a numbered rollback (`rollback 1` through `rollback 49`).

```text
! Operational mode -> Configuration mode
configure                      ! Shared candidate
configure private              ! Private candidate (recommended for multi-user)

! View config in hierarchical or flat "set" format
show                           ! Inside config mode
show | display set             ! Flat set commands
show | compare                 ! Diff candidate vs active running config

! Validate, commit, and safety timer
commit check                   ! Syntax/semantic check without applying
commit                         ! Apply and stay in config mode
commit and-quit                ! Apply and return to operational mode
commit confirmed 5             ! Auto-rollback in 5 min unless confirmed with "commit"

! Discard uncommitted edits or roll back to previous commit
rollback 0                     ! Discard candidate changes
rollback 1                     ! Load previous committed config (then run "commit")
show system commit             ! List recent commits with timestamps and users
```

## Interfaces and Addressing

### Cisco IOS-XE / NX-OS

```text
configure terminal
! Routed interface (add "no switchport" on L2/L3 switches and NX-OS)
interface GigabitEthernet1
 description to-core1
 no switchport
 ip address 10.1.12.1 255.255.255.252
 ipv6 address 2001:db8:12::1/64
 no shutdown
end
```

Verification:

```text
show ip interface brief
show ipv6 interface brief
show interfaces status
show interfaces GigabitEthernet1
show interfaces GigabitEthernet1 counters errors
show lldp neighbors
```

### Juniper Junos

```text
configure
set interfaces ge-0/0/0 description "to-core1"
set interfaces ge-0/0/0 unit 0 family inet address 10.1.12.1/30
set interfaces ge-0/0/0 unit 0 family inet6 address 2001:db8:12::1/64
set interfaces lo0 unit 0 family inet address 10.255.0.1/32
commit and-quit
```

Verification:

```text
show interfaces terse
show interfaces ge-0/0/0 detail
show interfaces ge-0/0/0 extensive
show lldp neighbors
show arp
show ipv6 neighbors
```

## VLANs and Layer 2 Switching

### Cisco IOS-XE / NX-OS

```text
configure terminal
vlan 10
 name USERS
!
interface Ethernet1/1
 switchport mode access
 switchport access vlan 10
!
interface Ethernet1/2
 switchport mode trunk
 switchport trunk allowed vlan 10,20
!
interface Vlan10
 ip address 10.10.10.1/24
 no shutdown
end
```

Verification: `show vlan brief`, `show interfaces trunk`, `show mac address-table`, `show spanning-tree`, `show port-channel summary` (NX-OS) / `show etherchannel summary` (IOS-XE).

### Juniper Junos (ELS syntax)

```text
configure
set vlans USERS vlan-id 10
set vlans USERS l3-interface irb.10
set interfaces ge-0/0/1 unit 0 family ethernet-switching interface-mode access
set interfaces ge-0/0/1 unit 0 family ethernet-switching vlan members USERS
set interfaces ge-0/0/2 unit 0 family ethernet-switching interface-mode trunk
set interfaces ge-0/0/2 unit 0 family ethernet-switching vlan members [ USERS SERVERS ]
set interfaces irb unit 10 family inet address 10.10.10.1/24
commit and-quit
```

Verification: `show vlans`, `show ethernet-switching interfaces`, `show ethernet-switching table`, `show spanning-tree interface`, `show lacp interfaces`.

## VRFs and Routing Instances

### Cisco IOS-XE

```text
configure terminal
vrf definition RED
 rd 65001:100
 address-family ipv4
 exit-address-family
!
interface GigabitEthernet2
 vrf forwarding RED
 ip address 10.20.1.1 255.255.255.0
end
```

### Cisco NX-OS

```text
configure terminal
vrf context RED
 rd 65001:100
!
interface Ethernet1/3
 no switchport
 vrf member RED
 ip address 10.20.1.1/24
end
```

Verification (IOS-XE & NX-OS): `show vrf`, `show ip route vrf RED`, `show ip arp vrf RED`, `ping vrf RED 10.20.1.2`.

### Juniper Junos

```text
configure
set routing-instances RED instance-type virtual-router
set routing-instances RED interface ge-0/0/3.0
set routing-instances RED routing-options static route 0.0.0.0/0 next-hop 10.20.1.254
commit and-quit
```

Verification: `show route instance`, `show route table RED.inet.0`, `ping routing-instance RED 10.20.1.2`.

## OSPF

### Cisco IOS-XE / NX-OS

```text
! IOS-XE
configure terminal
router ospf 1
 router-id 10.255.0.1
 passive-interface Loopback0
!
interface GigabitEthernet1
 ip ospf 1 area 0
 ip ospf network point-to-point
end
```

```text
! NX-OS (requires "feature ospf")
configure terminal
feature ospf
router ospf UNDERLAY
 router-id 10.255.0.1
!
interface Ethernet1/1
 ip router ospf UNDERLAY area 0.0.0.0
 ip ospf network point-to-point
end
```

Verification: `show ip ospf neighbor`, `show ip ospf interface brief`, `show ip ospf database`, `show ip route ospf`.

### Juniper Junos

```text
configure
set routing-options router-id 10.255.0.1
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 interface-type p2p
set protocols ospf area 0.0.0.0 interface lo0.0 passive
commit and-quit
```

Verification: `show ospf neighbor`, `show ospf interface`, `show ospf database`, `show route protocol ospf`.

## BGP

### Cisco IOS-XE / NX-OS

```text
configure terminal
ip prefix-list PL-LO seq 10 permit 10.255.0.1/32
!
route-map RM-OUT permit 10
 match ip address prefix-list PL-LO
!
router bgp 65001
 bgp router-id 10.255.0.1
 neighbor 10.1.12.2 remote-as 65002
 !
 address-family ipv4 unicast
  network 10.255.0.1 mask 255.255.255.255
  neighbor 10.1.12.2 activate
  neighbor 10.1.12.2 soft-reconfiguration inbound
  neighbor 10.1.12.2 route-map RM-OUT out
 exit-address-family
end
```

Verification:

```text
show bgp ipv4 unicast summary
show bgp ipv4 unicast neighbors 10.1.12.2
show bgp ipv4 unicast 203.0.113.0/24
show bgp ipv4 unicast neighbors 10.1.12.2 received-routes
show bgp ipv4 unicast neighbors 10.1.12.2 advertised-routes
clear bgp ipv4 unicast 10.1.12.2 soft in
```

### Juniper Junos

> [!NOTE]
> Junos exports **nothing** to BGP peers by default unless an `export` policy explicitly accepts routes (such as `protocol direct` for loopbacks).

```text
configure
set routing-options autonomous-system 65001
set policy-options policy-statement EXPORT-LO term 10 from protocol direct
set policy-options policy-statement EXPORT-LO term 10 from route-filter 10.255.0.1/32 exact
set policy-options policy-statement EXPORT-LO term 10 then accept
set protocols bgp group EBGP type external
set protocols bgp group EBGP export EXPORT-LO
set protocols bgp group EBGP neighbor 10.1.12.2 peer-as 65002
commit and-quit
```

Verification:

```text
show bgp summary
show bgp neighbor 10.1.12.2
show route protocol bgp
show route protocol bgp 203.0.113.0/24 extensive
show route receive-protocol bgp 10.1.12.2
show route advertising-protocol bgp 10.1.12.2
clear bgp neighbor 10.1.12.2 soft-inbound
```

## EVPN and VXLAN Inspection

| Task | Cisco NX-OS | Juniper Junos |
|---|---|---|
| EVPN BGP summary | `show bgp l2vpn evpn summary` | `show bgp summary` |
| EVPN RIB | `show bgp l2vpn evpn` | `show route table bgp.evpn.0` |
| VTEP peers | `show nve peers` | `show evpn database` |
| VNI mappings | `show nve vni` | `show evpn instance extensive` |
| Learned EVPN MACs | `show l2route evpn mac all` | `show evpn mac-table` |

## Everyday Troubleshooting

| Task | Cisco IOS-XE / NX-OS | Juniper Junos |
|---|---|---|
| Ping with source & DF bit | `ping 10.255.0.2 source Lo0 size 1500 df-bit` | `ping 10.255.0.2 source 10.255.0.1 size 1472 do-not-fragment` |
| Traceroute | `traceroute 10.255.0.2 numeric` | `traceroute 10.255.0.2 no-resolve` |
| Longest-match FIB check | `show ip cef 192.0.2.1` (IOS-XE) / `show forwarding route 192.0.2.1` (NX-OS) | `show route forwarding-table destination 192.0.2.1` |
| Control-plane packet capture | `ethanalyzer local interface inband display-filter "bgp"` (NX-OS) | `monitor traffic interface ge-0/0/0 no-resolve matching "port 179"` |
| ACL / firewall hit counts | `show access-lists` | `show firewall filter <name>` |
| System logs | `show logging last 50` | `show log messages \| last 50` |

See the [Lab and Network OS Command Reference](01-commands.md) to translate any of these commands directly to Linux, FRR, Arista EOS, or Nokia SR Linux.
