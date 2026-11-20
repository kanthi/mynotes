---
title: Intro
---

# Intro

**Lab Workbook** — Practice every core concept with 65+ structured, multi-NOS Containerlab and Linux production exercises spanning FRR, Arista cEOS, Nokia SR Linux, and Linux hosts.

## How to use this part

Start with [Workbook Guide](00-workbook-guide.md) to pick a learning path based on the images available on your lab host.

For short exercises without container images, compare [IP fragment offsets](68-ipv4-fragments-on-loopback.md), [TCP and UDP receive buffers](69-tcp-and-udp-short-reads.md), and [certificate trust and service names](70-certificate-path-and-service-name.md).

## Chapters in this part

| Chapter | File |
|---------|------|
| [Workbook Guide and Learning Paths](00-workbook-guide.md) | `00-workbook-guide.md` |
| [Lab Host and the Image Set](01-lab-host-and-images.md) | `01-lab-host-and-images.md` |
| [Three-Router Forwarding](02-three-router-forwarding.md) | `02-three-router-forwarding.md` |
| [cEOS Interfaces and LLDP](03-ceos-interfaces-and-lldp.md) | `03-ceos-interfaces-and-lldp.md` |
| [VLANs and Trunks on cEOS](04-vlans-and-trunks.md) | `04-vlans-and-trunks.md` |
| [Spanning Tree on cEOS](05-spanning-tree.md) | `05-spanning-tree.md` |
| [Link Aggregation on cEOS](06-link-aggregation.md) | `06-link-aggregation.md` |
| [Inter-VLAN Routing and DHCP](07-inter-vlan-and-dhcp.md) | `07-inter-vlan-and-dhcp.md` |
| [First Hop: VRRP and VARP](08-vrrp-and-varp.md) | `08-vrrp-and-varp.md` |
| [Static Routes across Three NOSes](09-static-routes.md) | `09-static-routes.md` |
| [OSPF in One Area](10-ospf-one-area.md) | `10-ospf-one-area.md` |
| [OSPF Areas, Cost, and BFD](11-ospf-areas-cost-bfd.md) | `11-ospf-areas-cost-bfd.md` |
| [IS-IS across FRR and cEOS](12-isis.md) | `12-isis.md` |
| [eBGP between Three Autonomous Systems](13-ebgp.md) | `13-ebgp.md` |
| [iBGP and a Route Reflector](14-ibgp-route-reflector.md) | `14-ibgp-route-reflector.md` |
| [BGP Policy](15-bgp-policy.md) | `15-bgp-policy.md` |
| [ACLs and Edge NAT](16-acls-and-nat.md) | `16-acls-and-nat.md` |
| [IPv6 on the Same Triangle](17-ipv6.md) | `17-ipv6.md` |
| [VXLAN Bridging](18-vxlan-bridge.md) | `18-vxlan-bridge.md` |
| [EVPN Control Plane for VXLAN](19-evpn-vxlan.md) | `19-evpn-vxlan.md` |
| [Reading SR Linux with gNMI](20-gnmi-srlinux.md) | `20-gnmi-srlinux.md` |
| [Production Failure Drills](21-production-failure-drills.md) | `21-production-failure-drills.md` |
| [Path MTU and TCP MSS](22-mtu-and-mss.md) | `22-mtu-and-mss.md` |
| [STP Edge Guards](23-stp-edge-guards.md) | `23-stp-edge-guards.md` |
| [DHCP Snooping and ARP Inspection](24-dhcp-snooping-and-dai.md) | `24-dhcp-snooping-and-dai.md` |
| [MLAG on cEOS](25-mlag.md) | `25-mlag.md` |
| [VRF-Lite](26-vrf-lite.md) | `26-vrf-lite.md` |
| [OSPF Stub, NSSA, and Summary](27-ospf-stub-and-summary.md) | `27-ospf-stub-and-summary.md` |
| [Redistribution and Route Tags](28-redistribute-and-tag.md) | `28-redistribute-and-tag.md` |
| [BGP MED, Prepend, and ECMP](29-bgp-med-prepend-ecmp.md) | `29-bgp-med-prepend-ecmp.md` |
| [Policy-Based Routing](30-policy-routing.md) | `30-policy-routing.md` |
| [Unicast RPF](31-urpf.md) | `31-urpf.md` |
| [SPAN](32-span.md) | `32-span.md` |
| [GRE over the Underlay](33-gre.md) | `33-gre.md` |
| [Control-Plane ACL](34-control-plane-acl.md) | `34-control-plane-acl.md` |
| [IPv6 SLAAC](35-ipv6-slaac.md) | `35-ipv6-slaac.md` |
| [NTP and Syslog](36-ntp-and-syslog.md) | `36-ntp-and-syslog.md` |
| [DSCP Trust and Measured Delay](37-dscp-and-delay.md) | `37-dscp-and-delay.md` |
| [OSPF Authentication](38-ospf-authentication.md) | `38-ospf-authentication.md` |
| [Multicast Distribution Failure Exercises](39-multicast-failure-exercises.md) | `39-multicast-failure-exercises.md` |
| [Policy Routing: Table Order, Throw and Blackhole](40-policy-lookup-and-throw.md) | `40-policy-lookup-and-throw.md` |
| [Address Planning, VLSM and Summary Failure](41-address-planning-and-summary.md) | `41-address-planning-and-summary.md` |
| [Build a Router and Break the Return Path](42-isolated-forwarding-return-path.md) | `42-isolated-forwarding-return-path.md` |
| [IPv6 Neighbor Discovery and an ICMPv6 Filter Failure](43-ipv6-neighbor-discovery-filter.md) | `43-ipv6-neighbor-discovery-filter.md` |
| [VLAN Access Membership: Build, Break and Repair](44-vlan-access-membership.md) | `44-vlan-access-membership.md` |
| [Tagged Trunk: A Missing VLAN and an Unaffected Control](45-tagged-trunk-failure.md) | `45-tagged-trunk-failure.md` |
| [Stateful Filtering: Direction, Counters and Return Traffic](46-stateful-acl-direction.md) | `46-stateful-acl-direction.md` |
| [Source NAT: Observe the Translation and Return Path](47-source-nat-observe.md) | `47-source-nat-observe.md` |
| [Destination NAT: Map a Service and Verify the Backend](48-destination-nat-observe.md) | `48-destination-nat-observe.md` |
| [DHCPv4 on the Wire: A Minimal Isolated Exchange](49-dhcpv4-wire-exchange.md) | `49-dhcpv4-wire-exchange.md` |
| [Linux VRFs: Overlapping Addresses and Lookup Fallback](50-vrf-overlap-and-fallback.md) | `50-vrf-overlap-and-fallback.md` |
| [ECMP: Observe Real Flows and Remove a Path](51-ecmp-real-flows.md) | `51-ecmp-real-flows.md` |
| [Classic STP: Observe a Triangle Reconfigure](52-classic-stp-triangle.md) | `52-classic-stp-triangle.md` |
| [LACP Bond: Member Failure and Recovery](53-lacp-member-failure.md) | `53-lacp-member-failure.md` |
| [eBGP Without Containers: Advertise and Withdraw a Route](54-ebgp-gobgp-namespaces.md) | `54-ebgp-gobgp-namespaces.md` |
| [eBGP Import Policy: Exact Prefixes and More-Specifics](55-ebgp-import-policy.md) | `55-ebgp-import-policy.md` |
| [Lab 56: Dual-ISP eBGP Graceful Shutdown (RFC 8326) and Prefix Circuit Breaker](56-bgp-dual-isp-graceful-shutdown.md) | `56-bgp-dual-isp-graceful-shutdown.md` |
| [Lab 57: OSPF Max-Metric (RFC 6987) Zero-Loss Maintenance Drain](57-ospf-max-metric-zero-loss-drain.md) | `57-ospf-max-metric-zero-loss-drain.md` |
| [Lab 58: BGP ADD-PATH (RFC 7911) on an iBGP Route Reflector](58-bgp-add-path-route-reflector.md) | `58-bgp-add-path-route-reflector.md` |
| [Lab 59: EVPN-VXLAN Symmetric IRB (L3VNI 10099) and Border-Leaf Type-5 Fabric](59-evpn-symmetric-irb-type5-fabric.md) | `59-evpn-symmetric-irb-type5-fabric.md` |
| [Lab 60: BGP Unnumbered Clos Fabric (RFC 8950)](60-bgp-unnumbered-clos-rfc8950.md) | `60-bgp-unnumbered-clos-rfc8950.md` |
| [Lab 61: Anycast DNS with BGP ECMP and Health-Checked Withdrawal](61-anycast-dns-ecmp-healthcheck.md) | `61-anycast-dns-ecmp-healthcheck.md` |
| [Lab 62: L4 Direct Server Return (DSR) Load Balancer with IPVS and BGP](62-l4-dsr-ipvs-bgp-load-balancer.md) | `62-l4-dsr-ipvs-bgp-load-balancer.md` |
| [Lab 63: Stateful Firewall HA with Keepalived (VRRP) and conntrackd](63-stateful-ha-keepalived-conntrackd.md) | `63-stateful-ha-keepalived-conntrackd.md` |
| [Lab 64: Route-Based IPsec (XFRM) Hybrid WAN with eBGP and MSS Clamping](64-ipsec-xfrm-bgp-hybrid-wan.md) | `64-ipsec-xfrm-bgp-hybrid-wan.md` |
| [Lab 65: Destination RTBH and BGP FlowSpec DDoS Mitigation](65-rtbh-and-bgp-flowspec-mitigation.md) | `65-rtbh-and-bgp-flowspec-mitigation.md` |
| [Lab 66: Streaming Telemetry with Nokia SR Linux, gNMIc, and Prometheus](66-streaming-telemetry-gnmic-prometheus.md) | `66-streaming-telemetry-gnmic-prometheus.md` |
| [Lab 67: eBPF XDP Packet Drop, TCP BBR, and fq_codel Bufferbloat Tuning](67-ebpf-xdp-drop-and-tcp-bbr-netem.md) | `67-ebpf-xdp-drop-and-tcp-bbr-netem.md` |
| [Lab 68 — Read IPv4 Fragment Offsets on Loopback](68-ipv4-fragments-on-loopback.md) | `68-ipv4-fragments-on-loopback.md` |
| [Lab 69 — Short Reads on TCP and UDP Sockets](69-tcp-and-udp-short-reads.md) | `69-tcp-and-udp-short-reads.md` |
| [Lab 70 — Verify the Certificate Path and Service Name](70-certificate-path-and-service-name.md) | `70-certificate-path-and-service-name.md` |
