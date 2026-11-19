---
title: Intro
---

# Intro

**Overlays and Tunnels** — Connect sites and decouple logical topologies from physical underlays using WireGuard, OpenVPN DCO, IPsec/IKEv2, VXLAN, EVPN, Tailscale, NAT traversal (STUN/TURN), MASQUE, and MPLS/L3VPN.

## How to use this part

Start with `01-why-encapsulate.md`, then choose your tunnel family: WireGuard/IPsec (`02`–`02g`), VXLAN/EVPN (`03`–`06b`), NAT traversal/MASQUE (`08`–`09`), or MPLS (`10`–`10b`).

## Chapters in this part

| Chapter | File |
|---------|------|
| [Why Encapsulate](01-why-encapsulate.md) | `01-why-encapsulate.md` |
| [Site Tunnels](02-site-tunnels.md) | `02-site-tunnels.md` |
| [WireGuard: Cryptokey Routing and the Pre-Shared Key](02b-wireguard-cryptokey-routing-and-psk.md) | `02b-wireguard-cryptokey-routing-and-psk.md` |
| [OpenVPN Data Channel Offload (DCO)](02c-openvpn-data-channel-offload.md) | `02c-openvpn-data-channel-offload.md` |
| [WireGuard Multi-Tunnel: AllowedIPs, Policy Routing, Kill Switch](02d-wireguard-multi-tunnel.md) | `02d-wireguard-multi-tunnel.md` |
| [WireGuard and MTU: Where the 1420 Comes From, Padding, and Silent Fragmentation](02e-wireguard-mtu-and-fragmentation.md) | `02e-wireguard-mtu-and-fragmentation.md` |
| [IPsec by Hand: ESP with ip xfrm, and What IKEv2 Automates](02f-ipsec-esp-xfrm-and-ikev2.md) | `02f-ipsec-esp-xfrm-and-ikev2.md` |
| [DHCP Option 121 and VPN Route Leaks (TunnelVision)](02g-dhcp-option-121-vpn-route-leaks.md) | `02g-dhcp-option-121-vpn-route-leaks.md` |
| [Route-Based IPsec: XFRM Interfaces, VTI and eBGP Failover](02h-route-based-ipsec-vti-xfrm-with-bgp.md) | `02h-route-based-ipsec-vti-xfrm-with-bgp.md` |
| [VXLAN Data Plane](03-vxlan-data-plane.md) | `03-vxlan-data-plane.md` |
| [EVPN Control-Plane Intro](04-evpn-control-plane-intro.md) | `04-evpn-control-plane-intro.md` |
| [EVPN Multihoming: Ethernet Segments, DF Election, Split Horizon and Aliasing](04b-evpn-multihoming-esi-df-election.md) | `04b-evpn-multihoming-esi-df-election.md` |
| [EVPN Symmetric IRB and Type-5 IP Prefix Routes](04c-evpn-symmetric-irb-and-type5-routes.md) | `04c-evpn-symmetric-irb-and-type5-routes.md` |
| [Overlay Lab](05-overlay-lab.md) | `05-overlay-lab.md` |
| [Head Office and Branch Office](06-tailscale-vxlan-sites.md) | `06-tailscale-vxlan-sites.md` |
| [Shared Address Space (100.64.0.0/10), CGNAT and Overlay Address Collisions](06b-shared-address-space-overlay-collisions.md) | `06b-shared-address-space-overlay-collisions.md` |
| [Multicast Essentials: IGMP, MLD, PIM and Snooping](07-multicast-essentials.md) | `07-multicast-essentials.md` |
| [NAT Traversal: STUN, Hole Punching and When You Need a Relay](08-nat-traversal-stun-hole-punching.md) | `08-nat-traversal-stun-hole-punching.md` |
| [TURN Relays: Credentials, Permissions and Peer Policy](08b-turn-relays-and-peer-policy.md) | `08b-turn-relays-and-peer-policy.md` |
| [MASQUE: Tunnelling UDP and IP over HTTP (CONNECT-UDP and CONNECT-IP)](09-masque-connect-udp-ip.md) | `09-masque-connect-udp-ip.md` |
| [MPLS Label Switching: Label Stacks, LDP, PHP and SR-MPLS on Linux](10-mpls-label-switching.md) | `10-mpls-label-switching.md` |
| [BGP/MPLS IP VPNs: VRFs, Route Distinguishers and Route Targets](10b-bgp-mpls-l3vpn.md) | `10b-bgp-mpls-l3vpn.md` |
