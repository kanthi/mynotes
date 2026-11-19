---
title: Lab and Network OS Command Reference
---

# Lab and Network OS Command Reference

This page is a side-by-side translation matrix for everyday inspection, configuration, and troubleshooting across six environments: **Linux (`ip`/`ss`/`nft`)**, **FRRouting (`vtysh`)**, **Arista EOS (`cEOS`/`vEOS`)**, **Nokia SR Linux**, **Cisco IOS-XE / NX-OS**, and **Juniper Junos**.

> [!NOTE]
> Exact keywords, interface naming (`eth1`, `Ethernet1`, `ethernet-1/1`, `GigabitEthernet1`, `ge-0/0/0`), and default VRF names (`default`, `vrf1`) depend on the image and release. Substitute the interface, VRF, or network-instance name from your topology.

## Dedicated Cheat Sheets

| Platform | Guide | Best for |
|---|---|---|
| FRRouting (FRR) | [FRRouting Cheat Sheet](02-frr.md) | `vtysh`, `/etc/frr/daemons`, OSPF, IS-IS, BGP, EVPN, BFD, Containerlab persistence |
| Arista EOS / cEOS / vEOS | [Arista EOS, cEOS, and vEOS Cheat Sheet](03-arista-eos.md) | Switching, MLAG, VXLAN/EVPN, `configure session`, `FastCli` / bash |
| Nokia SR Linux | [Nokia SR Linux Cheat Sheet](04-sr-linux.md) | Candidate datastore, `network-instance`, EVPN, gNMI, `srbase-default` netns |
| Lab tools & Linux | [Containerlab, netlab, and Linux Cheat Sheet](05-lab-tools.md) | `containerlab`, `netlab`, `ip`, `bridge`, `ss`, `tc`, `nft`, `tcpdump`, `iperf3` |
| Cisco & Juniper | [Cisco IOS-XE, NX-OS, and Juniper Junos Quick Reference](06-cisco-junos-quickref.md) | Mixed-vendor environments, IOS-XE/NX-OS CLI, Junos `set`/commit/rollback |

## System, Version, and Configuration

| Task | Linux | FRR (`vtysh`) | Arista EOS | Nokia SR Linux | Cisco IOS-XE / NX-OS | Juniper Junos |
|---|---|---|---|---|---|---|
| OS / software version | `uname -a; cat /etc/os-release` | `show version` | `show version` | `show version` | `show version` | `show version` |
| Uptime & hardware/daemons | `uptime; ps aux` | `show daemons` | `show version detail` | `show system information` | `show processes cpu` | `show system uptime` |
| Running configuration | `/etc/` files | `show running-config` | `show running-config` | `info from running` | `show running-config` | `show configuration` |
| Config in flat/set format | N/A | `show running-config` | `show running-config` | `info flat from running` | `show running-config` | `show configuration \| display set` |
| Logs | `journalctl -xe` | `show logging` | `show logging` | `show system logging buffer` | `show logging` | `show log messages` |

## Interfaces, Addressing, and Counters

| Task | Linux | FRR (`vtysh`) | Arista EOS | Nokia SR Linux | Cisco IOS-XE / NX-OS | Juniper Junos |
|---|---|---|---|---|---|---|
| Interface status summary | `ip -br link` | `show interface brief` | `show interfaces status` | `show interface brief` | `show interfaces status` | `show interfaces terse` |
| IPv4 addresses summary | `ip -br -4 addr` | `show interface brief` | `show ip interface brief` | `show interface all` | `show ip interface brief` | `show interfaces terse` |
| IPv6 addresses summary | `ip -br -6 addr` | `show interface` | `show ipv6 interface brief` | `show interface all` | `show ipv6 interface brief` | `show interfaces terse` |
| Detailed interface & MTU | `ip -d link show eth1` | `show interface eth1` | `show interfaces Ethernet1` | `show interface ethernet-1/1 detail` | `show interface Gi1` | `show interfaces ge-0/0/0 extensive` |
| Interface counters / errors | `ip -s -s link show eth1` | `show interface eth1` | `show interfaces Ethernet1 counters errors` | `show interface ethernet-1/1 statistics` | `show interface Gi1 counters errors` | `show interfaces ge-0/0/0 statistics` |
| Clear counters | Reboot / driver reset | `clear interface counters` | `clear counters Ethernet1` | `tools interface ethernet-1/1 statistics clear` | `clear counters Gi1` | `clear interfaces statistics ge-0/0/0` |

## VLANs, Layer 2, MAC Tables, and LLDP

| Task | Linux | FRR (`vtysh`) | Arista EOS | Nokia SR Linux | Cisco IOS-XE / NX-OS | Juniper Junos |
|---|---|---|---|---|---|---|
| VLAN database | `bridge vlan show` | N/A (Linux bridge) | `show vlan` | `show network-instance summary` | `show vlan brief` | `show vlans` |
| Trunk / tagged ports | `bridge vlan show` | N/A (Linux bridge) | `show interfaces trunk` | `info interface ethernet-1/1` | `show interfaces trunk` | `show ethernet-switching interfaces` |
| MAC address table | `bridge fdb show` | `show evpn mac vni all` | `show mac address-table` | `show network-instance <mac-vrf> bridge-table mac-table all` | `show mac address-table` | `show ethernet-switching table` |
| Spanning Tree (STP/RSTP) | `bridge link show` | N/A (Linux / `mstpd`) | `show spanning-tree` | `show network-instance <ni> protocols stp` | `show spanning-tree` | `show spanning-tree interface` |
| LAG / Port-Channel | `cat /proc/net/bonding/bond0` | N/A (Linux bonding/team) | `show port-channel summary` | `show lag` | `show etherchannel summary` / `show port-channel summary` | `show lacp interfaces` |
| LLDP neighbors | `lldpctl` | N/A (`lldpd`) | `show lldp neighbors` | `show system lldp neighbor` | `show lldp neighbors` | `show lldp neighbors` |

## ARP and IPv6 Neighbor Discovery

| Task | Linux | FRR (`vtysh`) | Arista EOS | Nokia SR Linux | Cisco IOS-XE / NX-OS | Juniper Junos |
|---|---|---|---|---|---|---|
| IPv4 ARP table | `ip -4 neigh show` | `show ip neigh` (via Linux) | `show arp` | `show arpnd arp-entries` | `show ip arp` | `show arp` |
| IPv6 Neighbor table | `ip -6 neigh show` | `show ipv6 neigh` (via Linux) | `show ipv6 neighbors` | `show arpnd neighbor-entries` | `show ipv6 neighbors` | `show ipv6 neighbors` |
| ARP in a VRF | `ip neigh show vrf RED` | `ip neigh show vrf RED` (Linux) | `show arp vrf RED` | `show network-instance RED neighbor` | `show ip arp vrf RED` | `show arp` (filtered by interface) |
| Flush neighbor cache | `ip neigh flush dev eth1` | `ip neigh flush dev eth1` (Linux) | `clear arp-cache` | `tools arpnd arp-entries clear` | `clear ip arp` | `clear arp` |

## IPv4/IPv6 Routing and VRFs

| Task | Linux | FRR (`vtysh`) | Arista EOS | Nokia SR Linux | Cisco IOS-XE / NX-OS | Juniper Junos |
|---|---|---|---|---|---|---|
| IPv4 routing table | `ip -4 route show` | `show ip route` | `show ip route` | `show network-instance default route-table ipv4-unicast summary` | `show ip route` | `show route table inet.0` |
| IPv6 routing table | `ip -6 route show` | `show ipv6 route` | `show ipv6 route` | `show network-instance default route-table ipv6-unicast summary` | `show ipv6 route` | `show route table inet6.0` |
| Longest-match lookup | `ip route get 192.0.2.1` | `show ip route 192.0.2.1` | `show ip route 192.0.2.1` | `show network-instance default route-table ipv4-unicast prefix 192.0.2.1/32` | `show ip route 192.0.2.1` | `show route 192.0.2.1` |
| List VRFs / instances | `ip vrf show` | `show vrf` | `show vrf` | `show network-instance summary` | `show vrf` | `show route instance` |
| VRF routing table | `ip route show vrf RED` | `show ip route vrf RED` | `show ip route vrf RED` | `show network-instance RED route-table ipv4-unicast summary` | `show ip route vrf RED` | `show route table RED.inet.0` |
| FIB / forwarding table | `ip route show` | `show ip fib` | `show ip route` / `show platform fap ip route` | `show network-instance default route-table ipv4-unicast summary` | `show ip cef` / `show forwarding route` | `show route forwarding-table` |

## OSPF

| Task | Linux | FRR (`vtysh`) | Arista EOS | Nokia SR Linux | Cisco IOS-XE / NX-OS | Juniper Junos |
|---|---|---|---|---|---|---|
| OSPF neighbors | N/A | `show ip ospf neighbor` | `show ip ospf neighbor` | `show network-instance default protocols ospf instance main neighbor` | `show ip ospf neighbor` | `show ospf neighbor` |
| OSPF interfaces | N/A | `show ip ospf interface` | `show ip ospf interface` | `show network-instance default protocols ospf instance main interface` | `show ip ospf interface brief` | `show ospf interface` |
| OSPF LSDB | N/A | `show ip ospf database` | `show ip ospf database` | `show network-instance default protocols ospf instance main database` | `show ip ospf database` | `show ospf database` |
| OSPF routes | N/A | `show ip ospf route` | `show ip route ospf` | `show network-instance default route-table ipv4-unicast route-type ospf` | `show ip route ospf` | `show route protocol ospf` |
| OSPFv3 (IPv6) neighbors | N/A | `show ipv6 ospf6 neighbor` | `show ipv6 ospf neighbor` | `show network-instance default protocols ospf instance v3 neighbor` | `show ospfv3 neighbor` | `show ospf3 neighbor` |

## BGP

| Task | Linux | FRR (`vtysh`) | Arista EOS | Nokia SR Linux | Cisco IOS-XE / NX-OS | Juniper Junos |
|---|---|---|---|---|---|---|
| BGP summary (IPv4) | N/A | `show bgp ipv4 unicast summary` | `show ip bgp summary` | `show network-instance default protocols bgp neighbor` | `show bgp ipv4 unicast summary` | `show bgp summary` |
| BGP summary (all AFs) | N/A | `show bgp summary` | `show bgp summary` | `show network-instance default protocols bgp neighbor` | `show bgp all summary` | `show bgp summary` |
| BGP neighbor detail | N/A | `show bgp neighbors <ip>` | `show ip bgp neighbors <ip>` | `show network-instance default protocols bgp neighbor <ip> detail` | `show bgp neighbors <ip>` | `show bgp neighbor <ip>` |
| BGP RIB / specific prefix | N/A | `show bgp ipv4 unicast <pfx>` | `show ip bgp <pfx>` | `show network-instance default protocols bgp routes ipv4 prefix <pfx>` | `show bgp ipv4 unicast <pfx>` | `show route protocol bgp <pfx> extensive` |
| Routes received from peer | N/A | `show bgp ipv4 unicast neighbors <ip> received-routes` | `show ip bgp neighbors <ip> received-routes` | `show network-instance default protocols bgp neighbor <ip> received-routes ipv4` | `show bgp ipv4 unicast neighbors <ip> received-routes` | `show route receive-protocol bgp <ip>` |
| Routes advertised to peer | N/A | `show bgp ipv4 unicast neighbors <ip> advertised-routes` | `show ip bgp neighbors <ip> advertised-routes` | `show network-instance default protocols bgp neighbor <ip> advertised-routes ipv4` | `show bgp ipv4 unicast neighbors <ip> advertised-routes` | `show route advertising-protocol bgp <ip>` |
| Soft reset inbound | N/A | `clear bgp <ip> soft in` | `clear ip bgp <ip> soft in` | `tools network-instance default protocols bgp neighbor <ip> reset-peer` | `clear bgp ipv4 unicast <ip> soft in` | `clear bgp neighbor <ip> soft-inbound` |

## EVPN and VXLAN

| Task | Linux | FRR (`vtysh`) | Arista EOS | Nokia SR Linux | Cisco NX-OS | Juniper Junos |
|---|---|---|---|---|---|---|
| EVPN BGP summary | N/A | `show bgp l2vpn evpn summary` | `show bgp evpn summary` | `show network-instance default protocols bgp neighbor` | `show bgp l2vpn evpn summary` | `show bgp summary` |
| EVPN route table | N/A | `show bgp l2vpn evpn` | `show bgp evpn` | `show network-instance default protocols bgp routes evpn summary` | `show bgp l2vpn evpn` | `show route table bgp.evpn.0` |
| VXLAN VNI / VTEP status | `ip -d link show type vxlan` | `show evpn vni` | `show vxlan vni` / `show vxlan vtep` | `show tunnel-interface vxlan0 vxlan-interface` | `show nve vni` / `show nve peers` | `show evpn database` |
| Remote VXLAN MACs | `bridge fdb show dev vxlan10` | `show evpn mac vni all` | `show vxlan address-table` | `show network-instance <mac-vrf> bridge-table mac-table all` | `show l2route evpn mac all` | `show evpn mac-table` |

## ACLs and Packet Filtering

| Task | Linux (`nftables`) | FRR (`vtysh`) | Arista EOS | Nokia SR Linux | Cisco IOS-XE / NX-OS | Juniper Junos |
|---|---|---|---|---|---|---|
| List active rules | `nft list ruleset` | `show access-lists` (control-plane) | `show ip access-lists` | `info acl` | `show access-lists` | `show firewall` |
| Rule hit counters | `nft list ruleset` (with `counter`) | N/A | `show ip access-lists` (with `counters per-entry`) | `show acl ipv4-filter <name> entry` | `show access-lists` | `show firewall filter <name>` |
| Clear ACL counters | `nft reset counters` | N/A | `clear ip access-list counters` | `tools acl ipv4-filter <name> statistics clear` | `clear access-list counters` | `clear firewall all` |

## Packet Capture and Live Diagnostics

| Task | Linux | FRR (`vtysh`) | Arista EOS | Nokia SR Linux | Cisco IOS-XE / NX-OS | Juniper Junos |
|---|---|---|---|---|---|---|
| Ping (source / VRF) | `ping -I eth1 -c 4 <ip>` | `ping <ip> vrf RED` | `ping vrf RED <ip> source Lo0` | `ping -I <src> network-instance RED <ip>` | `ping vrf RED <ip> source Lo0` | `ping routing-instance RED <ip> source <src>` |
| Traceroute | `traceroute -n <ip>` | `traceroute <ip>` | `traceroute vrf RED <ip>` | `traceroute network-instance RED <ip>` | `traceroute vrf RED <ip>` | `traceroute routing-instance RED <ip>` |
| Packet capture | `sudo tcpdump -ni eth1` | `sudo tcpdump -ni eth1` (Linux) | `tcpdump interface Ethernet1` | `bash sudo ip netns exec srbase-default tcpdump -ni e1-1` | `monitor capture` / `ethanalyzer` | `monitor traffic interface ge-0/0/0` |
| Listening sockets | `ss -tulpn` | `ss -tulpn` (Linux) | `bash ss -tulpn` | `bash sudo ip netns exec srbase-mgmt ss -tulpn` | `show control-plane host open-ports` | `show system connections` |

## Save, Commit, Diff, and Rollback

| Task | Linux | FRR (`vtysh`) | Arista EOS | Nokia SR Linux | Cisco IOS-XE / NX-OS | Juniper Junos |
|---|---|---|---|---|---|---|
| Enter config mode | Edit files / `ip` | `configure terminal` | `configure terminal` (or `configure session <name>`) | `enter candidate` | `configure terminal` | `configure` (or `configure private`) |
| Preview pending diff | `diff -u` | N/A (immediate) | `show session-config diffs` (in session) | `diff` | `show archive config differences` | `show \| compare` |
| Apply / commit | Reload service | Immediate on Enter | Immediate (or `commit` in session) | `commit now` / `commit stay` | Immediate (or `commit` on IOS-XR) | `commit` / `commit and-quit` |
| Confirmed commit (auto-revert) | N/A | N/A | `commit timer 00:05:00` (in session) | `commit confirmed timeout 300` | `configure terminal revert timer 5` | `commit confirmed 5` |
| Discard uncommitted changes | Restore file | N/A | `abort` (in session) | `discard now` | N/A | `rollback 0` |
| Rollback committed config | Restore backup | Restore `frr.conf` | `configure replace <file>` | `tools system configuration revert <id>` | `configure replace <url>` | `rollback 1` then `commit` |
| Save to startup / persist | Persist in netplan/systemd | `write memory` | `write memory` | `save startup` | `copy running-config startup-config` | Automatic on `commit` |

> [!TIP]
> Capture command output **before and after** every change so observed forwarding and control-plane state can be compared directly against your topology diagram and addressing plan. When tables do not explain a failure, take a `tcpdump` capture on both ends of the suspect link.
