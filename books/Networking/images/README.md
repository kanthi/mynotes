# Networking diagrams

**Style is monorepo-wide.** Do not invent a Networking-only palette.

| Doc | Path |
|-----|------|
| **Standard** | `includes/diagrams/STANDARD.md` |
| **Visual reference** | `includes/diagrams/reference/diagram-reference-topology.svg` (+ `-dark`) |

The hero figure `diagram-dist-switch-multihome.svg` (+ `-dark`) is the **first production exemplar** of that standard (same look as the monorepo reference).

## Inventory (this book)

| File | Chapter use | Dark sibling |
|------|-------------|--------------|
| `diagram-dist-switch-multihome.svg` | Index hero, L2 Ethernet, syllabus | yes |
| `diagram-mac-learning.svg` | Ethernet & MAC | yes |
| `diagram-vlan-trunk.svg` | VLANs & trunks | yes |
| `diagram-campus-vlan-vrrp-ospf.svg` | Campus lab: VLANs, VRRP, OSPF | yes |
| `diagram-lab-host-budget.svg` | Lab host image budget | yes |
| `diagram-multinos-triangle.svg` | FRR, cEOS, SR Linux triangle | yes |
| `diagram-ceos-vlan-trunk.svg` | cEOS VLANs and trunk | yes |
| `diagram-ceos-stp.svg` | cEOS spanning tree | yes |
| `diagram-ceos-lag.svg` | cEOS LACP | yes |
| `diagram-ceos-vrrp.svg` | cEOS VRRP | yes |
| `diagram-ospf-areas.svg` | OSPF areas on three NOSes | yes |
| `diagram-ebgp-triangle.svg` | eBGP triangle | yes |
| `diagram-ibgp-reflector.svg` | iBGP route reflector | yes |
| `diagram-vxlan-evpn.svg` | VXLAN and EVPN leaves | yes |
| `diagram-mtu-path.svg` | Path MTU | yes |
| `diagram-stp-guards.svg` | STP edge guards | yes |
| `diagram-dhcp-snoop.svg` | DHCP snooping | yes |
| `diagram-mlag.svg` | MLAG pair | yes |
| `diagram-vrf-lite.svg` | VRF-lite | yes |
| `diagram-ospf-stub.svg` | OSPF stub area | yes |
| `diagram-redist-tag.svg` | Redistribution tag | yes |
| `diagram-bgp-med.svg` | BGP MED | yes |
| `diagram-pbr.svg` | Policy routing | yes |
| `diagram-urpf.svg` | Unicast RPF | yes |
| `diagram-span.svg` | SPAN | yes |
| `diagram-gre.svg` | GRE tunnel | yes |
| `diagram-control-plane.svg` | Control-plane ACL | yes |
| `diagram-ipv6-slaac.svg` | IPv6 SLAAC | yes |
| `diagram-ntp-syslog.svg` | NTP and syslog | yes |
| `diagram-qos-delay.svg` | Measured delay | yes |
| `diagram-ospf-auth.svg` | OSPF MD5 | yes |
| `diagram-lab-triangle.svg` | Containerlab, statics | yes |
| `diagram-leaf-spine.svg` | Clos leaf-spine | yes |
| `diagram-tailscale-vxlan-sites.svg` | Head office / branch, Tailscale + VXLAN | yes |
| `diagram-k8s-packet-path.svg` | Kubernetes Pod-to-Service-to-Pod packet path | yes |
| `diagram-xdp-packet-path.svg` | Linux NIC → XDP → TC-BPF → Netfilter → Socket | yes |
| `diagram-l4-dsr-vs-l7.svg` | L4 Direct Server Return vs L7 reverse proxy | yes |
| `diagram-forensics-pipeline.svg` | TAP/SPAN → tcpdump → Zeek/Suricata → SIEM | yes |
| `diagram-tcp-state-machine.svg` | TCP lifecycle and UDP comparison | yes |
| `diagram-dns-recursive-resolution.svg` | DNS iterative resolution chain | yes |
| `diagram-multicast-pim-rpf.svg` | PIM-SM shared/source tree & RPF check | yes |
| `diagram-ospf-fsm-stuck-states.svg` | OSPF neighbor FSM & stuck-state diagnostics | yes |
| `diagram-bgp-decision-process.svg` | BGP best-path selection pipeline | yes |
| `diagram-bgp-add-path-vs-rr.svg` | BGP ADD-PATH vs Route Reflector path-hiding | yes |
| `diagram-igp-bgp-graceful-drain.svg` | Coordinated OSPF/IS-IS & BGP graceful drain | yes |
| `diagram-8021x-eapol-flow.svg` | 802.1X EAPOL & RADIUS authentication flow | yes |
| `diagram-qos-token-bucket-shaping.svg` | srTCM/trTCM token buckets, policing vs shaping | yes |
| `diagram-conntrack-nat-table.svg` | Linux nf_conntrack table & overflow drops | yes |
| `diagram-stateful-ha-conntrackd.svg` | Keepalived VRRP + conntrackd stateful HA | yes |
| `diagram-netops-sot-ci-pipeline.svg` | Source-of-Truth CI/CD & drift detection loop | yes |
| `diagram-ipsec-xfrm-bgp-ha.svg` | Route-based IPsec XFRM + active/active eBGP | yes |
| `diagram-evpn-symmetric-irb.svg` | EVPN Symmetric IRB L3VNI & Type-5 routing | yes |
| `diagram-cloud-vpc-tgw-peering.svg` | Multi-VPC Transit Gateway & Inspection VPC | yes |
| `diagram-k8s-bgp-cilium-ecmp.svg` | Kubernetes Cilium/MetalLB BGP ECMP & Egress GW | yes |
| `diagram-cdn-anycast-origin-shield.svg` | CDN Anycast Edge PoP & Origin Shield | yes |
| `diagram-service-mesh-sidecar-vs-ambient.svg` | Sidecar Envoy vs Ambient ztunnel/Waypoint mesh | yes |
| `diagram-bgp-forged-origin-hijack.svg` | Forged-origin BGP hijack bypassing RPKI ROV | yes |
| `diagram-bgp-otc-route-leak.svg` | RFC 9234 BGP Roles & OTC route-leak prevention | yes |
| `diagram-workbook-multicast-rpf.svg` | Lab 39: Multicast PIM-SM & RPF failure drills | yes |
| `diagram-lab-bgp-gshut-dual-isp.svg` | Lab 56: Dual-ISP BGP GSHUT & prefix limit | yes |
| `diagram-lab-ospf-max-metric-drain.svg` | Lab 57: OSPF max-metric zero-loss drain | yes |
| `diagram-lab-bgp-add-path.svg` | Lab 58: BGP ADD-PATH on iBGP Route Reflector | yes |
| `diagram-lab-evpn-symmetric-irb.svg` | Lab 59: EVPN Symmetric IRB & Type-5 fabric | yes |
| `diagram-lab-bgp-unnumbered-clos.svg` | Lab 60: BGP Unnumbered Clos (RFC 8950) | yes |
| `diagram-lab-anycast-ecmp-bgp.svg` | Lab 61: Anycast DNS with BGP ECMP health check | yes |
| `diagram-lab-l4-dsr-ipvs.svg` | Lab 62: L4 DSR load balancer with IPVS & BGP | yes |
| `diagram-lab-keepalived-conntrackd.svg` | Lab 63: Stateful firewall HA (VRRP + conntrackd) | yes |
| `diagram-lab-ipsec-xfrm-bgp.svg` | Lab 64: Route-based IPsec XFRM + eBGP WAN | yes |
| `diagram-lab-rtbh-flowspec.svg` | Lab 65: RTBH & BGP FlowSpec DDoS mitigation | yes |
| `diagram-lab-gnmic-prometheus.svg` | Lab 66: SR Linux gNMIc + Prometheus telemetry | yes |
| `diagram-lab-xdp-bbr-netem.svg` | Lab 67: eBPF XDP drop, TCP BBR & fq_codel | yes |
| `diagram-frame-signal-path.svg` | Physical signal path between Ethernet MACs | yes |
| `diagram-url-consumers.svg` | URL fields consumed by DNS, HTTP, and the browser | yes |
| `diagram-vm-move-sdn.svg` | Local and remote VM forwarding after a move | yes |
| `diagram-encryption-boundaries.svg` | Wi-Fi, VPN, and TLS endpoints on one path | yes |
| `diagram-portal-authorization.svg` | OIDC code flow and delegated API access | yes |
| `diagram-onion-circuit.svg` | Tor exit circuit and end-to-end HTTPS | yes |
| `diagram-map-request-return-path.svg` | Request across an exchange and reply via transit | yes |
| `diagram-map-peer-pieces.svg` | Directory contacts and independent map transfers | yes |
| `diagram-lab-ipv4-fragment-offsets.svg` | IPv4 fragment boundaries, sizes, and offsets | yes |
| `diagram-lab-short-socket-reads.svg` | Bytes remaining after short TCP and UDP receives | yes |
| `diagram-lab-certificate-decisions.svg` | One server certificate with three verifier inputs | yes |
| `diagram-onion-service-rendezvous.svg` | Separate Tor introduction and application circuits | yes |
| `diagram-encapsulation.svg` + 12 mono/concept SVGs | Models / habits | yes |

When adding illustrated figures: dual files + reference tokens + theme-swap (already in `styles/reader-mode-body.html`). Embed light path only.
