---
title: Arista EOS, cEOS, and vEOS Cheat Sheet
---

# Arista EOS, cEOS, and vEOS Cheat Sheet

Arista EOS runs a single Linux-based NOS across physical switches, containerized **cEOS-lab**, and virtual machine **vEOS-lab**. This cheat sheet covers everyday CLI workflows, Layer 2/Layer 3/EVPN configuration and verification, atomic commit sessions, and bash/`FastCli` integration.

## cEOS vs vEOS Platform Differences

| Aspect | cEOS-lab (Container) | vEOS-lab (VM) |
|---|---|---|
| Runtime | Docker / Containerlab (`kind: arista_ceos`) | KVM/QEMU (`vrnetlab` or standalone VM) |
| Boot time & footprint | Fast (~20–40s), ~1–2 GB RAM per node | Slower (~2–3 min), ~2–4 GB RAM per node |
| Interface naming | `Management0`, `Ethernet1`, `Ethernet2`... | `Management1`, `Ethernet1`, `Ethernet2`... |
| Data plane | Linux/software forwarding (`EosSdk` / software agent) | Software forwarding in VM kernel |
| Kernel / host sharing | Shares Linux host kernel (some sysctls/netns limits) | Dedicated guest Linux kernel |
| Common lab gotcha | Needs `ENV INTFTYPE=eth` (handled by Containerlab); MAC/LACP quirks on older releases | Watch serial console / vNIC driver (`virtio` vs `e1000`) |

> [!NOTE]
> For EVPN and modern BGP features on EOS, ensure multi-agent routing is enabled (`service routing protocols model multi-agent`). Changing the routing protocol model requires a reload.

## `Cli`, `bash`, and `FastCli`

You can enter the EOS CLI interactively, shell out to Linux bash from inside EOS, or run EOS commands non-interactively from bash or `docker exec` using `FastCli`.

```sh
# Interactive EOS CLI in Containerlab
docker exec -it clab-LAB-sw1 Cli

# Non-interactive EOS command from the host (great for automation & checks)
docker exec clab-LAB-sw1 FastCli -p 15 -c "show ip bgp summary"

# Multi-line config via FastCli
docker exec clab-LAB-sw1 FastCli -p 15 -c $'configure\ninterface Loopback0\nip address 10.255.0.1/32\nend'

# Enter Linux bash directly inside the container
docker exec -it clab-LAB-sw1 bash
```

Inside the EOS `Cli`:

```text
enable
bash                        ! Drop into Linux bash shell
bash ip -br addr            ! Run a single Linux command without leaving Cli
show version
show running-config
show running-config section bgp
show running-config interfaces Ethernet1
```

## Interfaces, SVIs, Trunks, LAG, and MLAG

### Routed interface, access port, trunk, and SVI

```text
configure terminal
! Routed (L3) interface
interface Ethernet1
   description to-spine1
   no switchport
   ip address 10.1.12.1/30
   ipv6 address 2001:db8:12::1/64
   no shutdown

! Access (L2) interface
interface Ethernet2
   switchport mode access
   switchport access vlan 10

! Trunk (802.1Q) interface
interface Ethernet3
   switchport mode trunk
   switchport trunk allowed vlan 10,20,30

! SVI (L3 gateway for VLAN 10)
vlan 10
   name USERS
interface Vlan10
   ip address 10.10.10.1/24
   no shutdown
end
```

### LAG (Port-Channel) and MLAG

```text
configure terminal
interface Ethernet4-5
   channel-group 10 mode active
!
interface Port-Channel10
   switchport mode trunk
   switchport trunk allowed vlan 10,20
   mlag 10
end
```

### Inspection commands

```text
show interfaces status
show ip interface brief
show ipv6 interface brief
show interfaces Ethernet1
show interfaces Ethernet1 counters errors
show interfaces trunk
show port-channel summary
show lacp interface
show mlag
show mlag interfaces
show lldp neighbors
```

## VLANs and Spanning Tree

```text
show vlan
show vlan brief
show mac address-table
show mac address-table vlan 10
show spanning-tree
show spanning-tree vlan 10
show spanning-tree root
show spanning-tree blockedports
```

## VRFs

EOS requires `ip routing` globally and `ip routing vrf <NAME>` per VRF before forwarding IPv4 traffic inside that VRF.

```text
configure terminal
ip routing
vrf instance RED
!
ip routing vrf RED
!
interface Ethernet2
   no switchport
   vrf RED
   ip address 10.20.1.1/24
end
```

Inspect VRF state:

```text
show vrf
show ip interface brief vrf RED
show ip route vrf RED
show arp vrf RED
ping vrf RED 10.20.1.2
```

## OSPF

```text
configure terminal
interface Loopback0
   ip address 10.255.0.1/32
   ip ospf area 0.0.0.0
!
interface Ethernet1
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
router ospf 1
   router-id 10.255.0.1
   passive-interface Loopback0
   max-lsa 12000
end
```

Inspect OSPF:

```text
show ip ospf
show ip ospf neighbor
show ip ospf interface brief
show ip ospf database
show ip route ospf
```

## BGP, Prefix-Lists, and Route-Maps

```text
configure terminal
ip prefix-list PL-LOOPBACK seq 10 permit 10.255.0.0/24 le 32
!
route-map RM-UNDERLAY-OUT permit 10
   match ip address prefix-list PL-LOOPBACK
!
router bgp 65001
   router-id 10.255.0.1
   no bgp default ipv4-unicast
   maximum-paths 4 ecmp 4
   neighbor 10.1.12.2 remote-as 65002
   neighbor 10.1.12.2 send-community extended
   !
   address-family ipv4
      neighbor 10.1.12.2 activate
      neighbor 10.1.12.2 route-map RM-UNDERLAY-OUT out
      network 10.255.0.1/32
end
```

Inspect BGP:

```text
show ip bgp summary
show bgp summary
show ip bgp neighbors 10.1.12.2
show ip bgp
show ip bgp 203.0.113.0/24
show ip bgp neighbors 10.1.12.2 received-routes
show ip bgp neighbors 10.1.12.2 advertised-routes
clear ip bgp 10.1.12.2 soft in
clear ip bgp 10.1.12.2 soft out
```

## VXLAN and EVPN

```text
configure terminal
service routing protocols model multi-agent
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vrf RED vni 50001
!
router bgp 65001
   neighbor EVPN-PEERS peer group
   neighbor EVPN-PEERS update-source Loopback0
   neighbor EVPN-PEERS ebgp-multihop 3
   neighbor EVPN-PEERS send-community extended
   neighbor 10.255.0.2 peer group EVPN-PEERS
   neighbor 10.255.0.2 remote-as 65000
   !
   vlan 10
      rd auto
      route-target both 10010:10010
      redistribute learned
   !
   address-family evpn
      neighbor EVPN-PEERS activate
   !
   vrf RED
      rd 10.255.0.1:50001
      route-target import evpn 50001:50001
      route-target export evpn 50001:50001
      redistribute connected
end
```

Inspect VXLAN and EVPN:

```text
show bgp evpn summary
show bgp evpn
show bgp evpn route-type mac-ip
show bgp evpn route-type imet
show bgp evpn route-type ip-prefix
show vxlan vtep
show vxlan vni
show vxlan address-table
show vxlan flood vtep
```

## ACLs and Control-Plane Filtering

```text
configure terminal
ip access-list ACL-IN
   counters per-entry
   10 permit tcp 10.10.10.0/24 any eq 443
   20 deny icmp any any
   30 permit ip any any
!
interface Ethernet2
   ip access-group ACL-IN in
end
```

Inspect and clear counters:

```text
show ip access-lists
show ip access-lists ACL-IN
clear ip access-list counters ACL-IN
```

## `tcpdump` and Monitor Sessions

EOS provides built-in `tcpdump` directly from the CLI for control-plane / CPU-punted packets (and all packets on cEOS/vEOS software interfaces), plus SPAN (`monitor session`) for mirroring hardware switch ports.

```text
! Live capture from EOS CLI
tcpdump interface Ethernet1 filter tcp port 179
tcpdump interface Ethernet1 verbose

! Port mirror (SPAN)
configure terminal
monitor session LAB-SPAN source Ethernet1 both
monitor session LAB-SPAN destination Ethernet6
end
show monitor session
```

## Atomic Changes with `configure session`

Unlike plain `configure terminal` (which applies lines immediately), `configure session` lets you stage changes, review the exact diff, validate syntax, commit atomically, or auto-rollback with a commit timer.

```text
configure session LAB-CHANGE
   interface Ethernet1
      description verified-change
   router bgp 65001
      neighbor 10.1.12.2 maximum-routes 1000
show session-config diffs
commit                       ! Apply atomically
! Or commit with a 5-minute safety rollback timer:
! commit timer 00:05:00      (confirm later with "configure session LAB-CHANGE commit")
! Or cancel without applying:
! abort
```

Rollback to a saved checkpoint or startup config:

```text
configure replace startup-config
```

## Saving Configuration

```text
write memory                 ! Equivalent to: copy running-config startup-config
show startup-config
diff running-config startup-config
```

> [!TIP]
> In Containerlab, `write memory` saves to `/mnt/flash/startup-config` inside the container. Use `containerlab save -t topology.clab.yml` or copy `/mnt/flash/startup-config` back to your repo before destroying the lab. See [Containerlab, netlab, and Linux Cheat Sheet](05-lab-tools.md).
