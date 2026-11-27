---
unnumbered: true
---

# Networking

Learn networking from first principles to NetOps by designing, building, breaking, and verifying networks. Containerlab and netlab support labs using freely available images, including FRR, Arista cEOS, and Nokia SR Linux.

![Distribution switch with multi-homed servers](images/diagram-dist-switch-multihome.svg)

## Read from the wire to a signed-in application

| Topic | Reading path | Question to answer |
|---|---|---|
| Foundations | [Nodes and interfaces](content/02-networking-foundations/01-what-is-networking.md) → [layered models and protocol contracts](content/02-networking-foundations/02-tcpip-and-layered-models.md) | Which device makes each forwarding decision, and what fields does it need? |
| Physical links and LANs | [Signals and NICs](content/02-networking-foundations/14-from-signals-to-frames.md) → [Ethernet switching](content/03-layer-2/01-ethernet-mac.md) → [VLANs and trunks](content/03-layer-2/02-vlans-trunks.md) | Where does a bit become a frame, and how far can a broadcast travel? |
| Addressing, routing, and conversations | [A packet across a router](content/02-networking-foundations/03-packets-frames-encapsulation.md) → [subnetting](content/02-networking-foundations/05-ipv4-addressing-and-subnetting.md) → [a connection across the Internet](content/04-ip-routing-services/10-follow-a-connection-across-the-internet.md) → [fragmentation and ICMP](content/04-ip-routing-services/04-icmp-path-mtu.md) → [TCP and UDP](content/02-networking-foundations/09-transport-tcp-udp.md) | Which addresses change at a hop, and which endpoint recovers missing data? |
| Applications and virtual networks | [Host configuration and DNS](content/02-networking-foundations/10-common-application-protocols.md) → [URLs and HTTP](content/02-networking-foundations/15-from-a-url-to-a-web-response.md) → [peer discovery and verified delivery](content/02-networking-foundations/16-peer-discovery-and-piece-verification.md) → [virtual interfaces, overlays, and SDN](content/07-a-overlays-tunnels/11-virtual-interfaces-and-sdn.md) | Who resolves a name, who serves a resource, and who installs the path? |
| Security and privacy | [Keys and encryption boundaries](content/06-policy-qos-hardening/12-keys-identities-and-encryption-boundaries.md) → [firewall decisions](content/06-policy-qos-hardening/07-nftables-fundamentals.md) → [federated identity and access](content/06-policy-qos-hardening/13-federated-login-and-api-authorization.md) → [onion routing](content/07-a-overlays-tunnels/12-onion-routing-and-observer-visibility.md) | Who can read the request, authenticate the user, or authorize the operation? |

## Learning path

1. [Lab platform](content/01-lab-platform/index.md) and a first topology on macOS or Linux.
2. [Networking foundations](content/02-networking-foundations/index.md) and packet capture.
3. [Layer 2](content/03-layer-2/index.md): switching, VLANs, loops, aggregation, and enterprise Wi-Fi.
4. [IP routing](content/04-ip-routing-services/index.md) and [network services](content/04-network-services/index.md): static routes, DNS, DHCP, NAT, redundancy, and MTU.
5. [Dynamic routing](content/05-dynamic-routing/index.md): OSPF, IS-IS, BGP, policy, Segment Routing, and convergence.
6. [Policy, QoS, and hardening](content/06-policy-qos-hardening/index.md) and [security operations / NetOps](content/06-security-operations/index.md): automation, Source of Truth, Batfish/Suzieq, and streaming telemetry.
7. [Overlays and tunnels](content/07-a-overlays-tunnels/index.md), [fabrics and multi-area](content/07-b-fabrics-multi-area/index.md), [cloud and Kubernetes networking](content/07-c-cloud-kubernetes-networking/index.md), and [programmable dataplanes and traffic delivery](content/07-d-dataplanes-and-traffic-delivery/index.md).
8. Optional advanced tracks: [BGP security](content/08-bgp-security/index.md), [network forensics](content/08-network-forensics/index.md), and [transport, TLS/PKI, and HTTP/proxies](content/08-transport-and-applications/index.md).
9. [Lab workbook](content/09-lab-workbook/index.md) (55+ multi-NOS exercises) and worked [capstone projects](content/09-projects/index.md) (C1–C4).
10. [Command reference](content/10-command-reference/index.md): side-by-side NOS translation matrix and cheat sheets for FRR, Arista EOS/cEOS/vEOS, Nokia SR Linux, Containerlab/netlab/Linux, and Cisco/Junos.
11. [Network automation with Ansible](content/11-network-automation-with-ansible/index.md): 34-chapter production track plus appendices covering Ansible core mechanics, connection plugins, Cisco/Junos/FortiOS resource modules, Jinja2, Source of Truth, compliance, testing, CI/CD, and AWX/AAP.

Use the [workbook guide and learning paths](content/09-lab-workbook/00-workbook-guide.md) to choose exercises and prerequisites. Use each diagram to predict the path, build the lab, verify the control and data planes, then introduce a failure and explain what changed. The workbook includes optional cEOS exercises that require an independently obtained image; consult the image provider for access and licensing. The workbook complements the core chapters: choose labs that match your topic and available images. Cisco and Juniper labs may be added as optional tracks later.

Every serious lab should include a topology and addressing plan, verification, and a failure scenario. Topology diagrams follow the shared illustrated light and dark SVG standard; conceptual diagrams explain the behavior being tested.
