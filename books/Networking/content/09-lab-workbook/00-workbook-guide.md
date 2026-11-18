---
title: Workbook Guide and Learning Paths
---

# Workbook Guide and Learning Paths

This workbook is a collection of experiments, not a requirement to install every network operating system. Start with the concepts and exercises that match your current knowledge and lab environment. Each path below moves from foundations toward more complex failures; repeat a prerequisite whenever its evidence is unclear. The book is vendor-neutral: platform-specific commands are examples of a concept, and their defaults and supported features must be checked on the chosen version.

## Before every lab

Read the topology, addressing, software prerequisites and cleanup instructions. Predict the expected path and outcome, capture a baseline, introduce one fault at a time, then verify the repair. Record device versions, timestamps, the observation point, route or VLAN state, counter deltas and the result in both directions. Treat simulated output or a written procedure as a hypothesis until you run it on the stated platform. Some exercises require licensed or separately obtained images; the individual chapter identifies these requirements.

Linux namespace labs can run without network-device images when the host permits unprivileged user/network namespaces and has the listed kernel features and tools. They alter only the isolated lab namespace when run exactly as instructed; do not copy their firewall, DHCP or interface commands into the host namespace. Containerlab and multi-NOS exercises need their specified images and a working container runtime. Hardware, wireless and managed-switch exercises require appropriate equipment or feature-capable simulation.

## Foundation path

| Order | Learn and practice | Evidence to retain |
|---|---|---|
| 1 | [IPv4 addressing](../02-networking-foundations/05-ipv4-addressing-and-subnetting.md) and [address planning](41-address-planning-and-summary.md) | Prefix, range, overlap and summary checks |
| 2 | [Ethernet and frames](../02-networking-foundations/04-link-layer-and-ethernet.md), then [access VLAN membership](44-vlan-access-membership.md) | Port VLAN table, ARP behavior and restored connectivity |
| 3 | [Tagged trunk failure](45-tagged-trunk-failure.md) | Affected VLAN, unaffected control VLAN and both trunk memberships |
| 4 | [IP forwarding and return path](42-isolated-forwarding-return-path.md) | Forward and reverse routes, loss and recovery |
| 5 | [IPv6 Neighbor Discovery](43-ipv6-neighbor-discovery-filter.md) | Route, neighbor state, filter counter and recovery |
| 6 | [DHCPv4 exchange](49-dhcpv4-wire-exchange.md) | Message types and transaction correlation; distinguish the teaching code from a production lease |
| 7 | [Stateful filtering](46-stateful-acl-direction.md), [source NAT](47-source-nat-observe.md) and [destination NAT](48-destination-nat-observe.md) | Policy counters, observed original/translated addresses and return path |

The namespace-based examples above were run on the writing host as documented in their chapters. Kernel, tool and permission differences can change the result; check the prerequisites and record your own observations.

## Switching and campus path

Study [VLANs and trunks](../03-layer-2/02-vlans-trunks.md), [spanning-tree elections](../03-layer-2/03b-spanning-tree-election.md) and [link aggregation](../03-layer-2/04-link-aggregation.md). Then build the workbook's [VLAN](04-vlans-and-trunks.md), [spanning tree](05-spanning-tree.md), [aggregation](06-link-aggregation.md) and [Linux LACP member failure](53-lacp-member-failure.md), [inter-VLAN and DHCP](07-inter-vlan-and-dhcp.md), and [first-hop redundancy](08-vrrp-and-varp.md) labs. Extend the [campus project](../09-projects/03b-campus-vlan-vrrp-ospf.md) with the [failure-domain matrix](../09-projects/03c-campus-failure-domain-design.md).

Checkpoint: for a chosen VLAN, identify the access port, trunk membership, active spanning-tree path, virtual gateway owner and route to the destination. Explain what changes when an uplink, gateway or access switch fails. Verify software support and exact topology before assuming a guard or redundancy behavior from another implementation.

## Routing and policy path

Begin with [static routes](09-static-routes.md) and the [policy-table lookup exercise](40-policy-lookup-and-throw.md) and [VRF overlap and fallback](50-vrf-overlap-and-fallback.md). Continue through [single-area OSPF](10-ospf-one-area.md), [areas and BFD](11-ospf-areas-cost-bfd.md), [IS-IS](12-isis.md), [eBGP](13-ebgp.md) or [eBGP with GoBGP](54-ebgp-gobgp-namespaces.md) followed by [import policy](55-ebgp-import-policy.md), [route reflectors](14-ibgp-route-reflector.md) and [BGP policy](15-bgp-policy.md). For advanced troubleshooting, use [OSPF adjacency diagnostics](../05-dynamic-routing/01b-ospf-adjacency-diagnostics.md), [redistribution design](../05-dynamic-routing/03b-redistribution-design.md), [route tags](28-redistribute-and-tag.md) and [MED, prepend and ECMP](29-bgp-med-prepend-ecmp.md), and [ECMP real flows](51-ecmp-real-flows.md).

Checkpoint: given a prefix, identify its origin, advertisement, import decision, eligible paths, selected RIB route, forwarding next hop and return path. Explain why a full adjacency or established session does not prove that a particular route is usable. Compare behavior against the actual NOS and version rather than a universal preference table.

## Services, security and operations path

Use [DHCP relay diagnostics](../04-network-services/02d-dhcp-lease-relay-diagnostics.md), [MTU and MSS](22-mtu-and-mss.md), [IPv6 operations](17-ipv6.md), [IPv6 first-hop security](../06-policy-qos-hardening/11-ipv6-first-hop-security.md), [QoS queues](../06-policy-qos-hardening/03b-qos-queues-and-token-buckets.md), and [API conditional updates](../06-security-operations/06b-api-conditional-updates.md). Add [ACL and NAT](16-acls-and-nat.md), [uRPF](31-urpf.md), [control-plane ACL](34-control-plane-acl.md), [NTP and syslog](36-ntp-and-syslog.md) and [DSCP and delay](37-dscp-and-delay.md) where supported.

Checkpoint: correlate a client symptom with an observation at the access edge, control plane and destination; distinguish an address-configuration failure from DNS, routing, firewall and application failures. For every proposed change, name the trust boundary, expected counter and rollback condition.

## Advanced overlay and integration path

After underlay routing works, progress through [VXLAN bridge](18-vxlan-bridge.md), [EVPN/VXLAN](19-evpn-vxlan.md), [GRE](33-gre.md), [VRF-lite](26-vrf-lite.md) and the [dual-home project](../09-projects/04-edge-dual-home-lab.md). Use [multicast distribution](../04-ip-routing-services/09-multicast-distribution.md) and its [failure exercises](39-multicast-failure-exercises.md) when the design needs multicast. Validate underlay next-hop reachability, encapsulation MTU, overlay control-plane state and tenant forwarding separately.

The paths are learning aids, not a guarantee of complete certification coverage. Consult the current objectives for the specific certification and platform, and practice any platform-specific tasks using their current documentation.
