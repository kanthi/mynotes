---
title: Container Networking and CNI Primitives
---

# Container Networking and CNI Primitives

Containers do not have virtual hardware NICs or hypervisors. A container's "network" is simply a Linux **network namespace** connected to the host (or another namespace) via kernel virtual interfaces such as `veth` pairs, bridges, MACVLAN, or IPVLAN. Kubernetes delegates the creation and wiring of these primitives to **Container Network Interface (CNI)** plugins.

## Learning goals

By the end of this chapter you can:

- Explain how Linux network namespaces, `veth` pairs, and bridges isolate and connect containers
- Compare Linux bridge, MACVLAN, and IPVLAN container attachment models
- Describe the CNI specification lifecycle (`ADD`, `DEL`, `CHECK`) and IPAM plugins
- Contrast overlay CNIs (VXLAN/Geneve) with native-routing / BGP CNIs (Calico, Cilium)
- Build and verify a working two-container network from scratch using `ip netns`

## Linux kernel building blocks

Every container runtime (containerd, CRI-O, Docker) relies on the same core Linux networking primitives:

### Network namespaces (`netns`)

A network namespace gives a set of processes its own isolated copy of:

- Network interfaces (starting with only a down `lo` interface)
- IPv4 and IPv6 routing tables (`ip route`)
- Neighbor / ARP tables (`ip neigh`)
- Firewall and NAT rules (`nftables`, `iptables`, `conntrack`)
- Socket tables (`ss -tulpn`) and `/proc/net`

In Kubernetes, all containers inside a **Pod** share a single network namespace (created by the `pause` sandbox container). That is why sidecars and app containers in the same Pod talk over `127.0.0.1` and share one Pod IP address.

### Virtual Ethernet (`veth`) pairs

A `veth` pair is a virtual patch cable with two ends: whatever frame enters one end immediately exits the other. To connect a container to the host:

1. Create a `veth` pair in the host root namespace (e.g., `veth-host` and `veth-pod`).
2. Move `veth-pod` into the container's network namespace and rename it `eth0`.
3. Leave `veth-host` in the root namespace and attach it to a Linux bridge or route directly to it via `/32` host routes.

### Attachment models: Bridge vs MACVLAN vs IPVLAN

| Model | How it works | Pros | Cons / caveats |
|-------|--------------|------|----------------|
| **veth + Linux bridge** | Pod `veth` attaches to a host L2 bridge (`cni0`/`docker0`); host routes out `eth0` | Universal compatibility; works with Netfilter, eBPF, and cloud VPCs | Extra virtual L2 hop and bridge FDB lookup |
| **veth + L3 point-to-point** | No bridge; host installs a `/32` route (`10.244.1.5 dev cali123`) directly to the `veth` | Eliminates bridge overhead; cleaner ARP/policy boundary (used by Calico & Cilium) | Relies on proxy ARP or `/32` link-local gateway (`169.254.1.1`) |
| **MACVLAN** | Sub-interfaces on host NIC, each with a unique MAC address | Near-bare-metal latency; bypasses host bridge | Cloud VPCs drop unknown MACs; host cannot talk to its own MACVLAN children without a workaround |
| **IPVLAN (L2/L3)** | Shares the host NIC's single MAC address across containers | Passes cloud/switch port-security MAC limits; very low overhead | L3 mode breaks DHCP/ARP/broadcast inside the container |

> [!NOTE]
> Most modern Kubernetes CNIs (Calico, Cilium, AWS VPC CNI) prefer **veth + direct L3 host routes** (or eBPF `bpf_redirect_peer`) instead of a Linux bridge. Skipping the bridge avoids unnecessary L2 header lookups and broadcast domain emulation on every packet.

## The CNI specification

**CNI (Container Network Interface)** is a minimal, runtime-agnostic contract defined by the CNCF. Rather than running a complex RPC protocol, the container runtime invokes a CNI binary on disk and passes JSON configuration on `stdin` plus parameters in environment variables (`CNI_COMMAND`, `CNI_CONTAINERID`, `CNI_NETNS`, `CNI_IFNAME`).

### CNI lifecycle verbs

| Verb | Triggered when | Responsibility |
|------|----------------|----------------|
| **`ADD`** | Pod sandbox starts | Create `veth`/interface, move into `CNI_NETNS`, invoke IPAM to allocate IP, configure routes/policy, return JSON result |
| **`DEL`** | Pod terminates | Release IP in IPAM, delete host-side `veth` and routes; must be idempotent |
| **`CHECK`** | Runtime health audit | Verify that the Pod's interface, IP, and routes still match expected state |
| **`VERSION`** | Capability discovery | Report supported CNI spec versions |

### Chained plugins and IPAM

CNI configurations (stored in `/etc/cni/net.d/*.conflist`) chain multiple plugins in sequence. For example, a primary plugin wires the interface and calls an **IPAM plugin** (`host-local`, `calico-ipam`, `cilium-ipam`, or cloud-native ENI IPAM) to lease an IP, and a second plugin (`portmap` or `bandwidth`) adds hostPort DNAT or traffic shaping:

```json
{
  "cniVersion": "1.0.0",
  "name": "k8s-pod-network",
  "plugins": [
    {
      "type": "bridge",
      "bridge": "cni0",
      "isGateway": true,
      "ipMasq": true,
      "ipam": {
        "type": "host-local",
        "subnet": "10.244.1.0/24",
        "routes": [{ "dst": "0.0.0.0/0" }]
      }
    },
    {
      "type": "portmap",
      "capabilities": { "portMappings": true }
    }
  ]
}
```

## Multi-node Pod networking: Overlay vs Native Routing

Kubernetes imposes one fundamental networking rule: **every Pod gets a unique IP address, and any Pod on any node must be able to reach any other Pod without NAT**. Two architectural families achieve this across worker nodes:

### 1. Overlay CNIs (VXLAN / Geneve / IPIP)

Each node runs a tunnel endpoint (VTEP). When Pod A (`10.244.1.5`) sends a packet to Pod B (`10.244.2.9`) on Node B, Node A encapsulates the original Pod packet inside a UDP VXLAN (port `4789`, 50-byte overhead) or Geneve (port `6081`) packet addressed to Node B's node IP (see [VXLAN Data Plane](../07-a-overlays-tunnels/03-vxlan-data-plane.md)).

- **Pros**: Works on any L3 underlay (cloud VPCs, bare-metal racks, multi-subnet clusters) with zero router configuration; Pod CIDRs are decoupled from underlay IP tables.
- **Cons**: Encapsulation reduces effective Pod MTU (e.g., `1450` on a `1500`-byte underlay); slightly higher CPU cost unless NICs support VXLAN/Geneve GRO/TSO offload.

### 2. Native-routing / BGP CNIs (Calico BGP, Cilium BGP, Cloud ENI CNIs)

Pod packets leave the node unencapsulated with their real Pod source and destination IPs. For the underlay to route them:

- **On-premises / Clos fabric**: Each node runs a BGP speaker (Calico BIRD, Cilium BGP Control Plane, or FRR) and peers with its Top-of-Rack (ToR) leaf switches, advertising its local Pod CIDR (e.g., `10.244.1.0/24`).
- **Same-L2 flat network**: Nodes peer with each other via BGP or install direct routes pointing to the peer node's IP as next-hop.
- **Cloud-native ENI mode (AWS VPC CNI, Azure CNI, GKE Alias IP)**: Pods receive real VPC subnet IP addresses directly from the cloud control plane. The VPC fabric routes Pod traffic natively with zero overlay overhead and full VPC Flow Log visibility—at the cost of higher VPC IP address consumption.

| Dimension | Overlay (VXLAN/Geneve) | Native Routing (BGP) | Cloud ENI / Alias IP |
|-----------|------------------------|----------------------|----------------------|
| **Encapsulation overhead** | 50+ bytes (MTU ~1450) | None (full 1500/9000 MTU) | None |
| **Underlay dependency** | None (any IP reachability) | BGP peering or cloud route tables | Cloud-specific VPC IPAM |
| **IP consumption** | Private overlay CIDR | Routable Pod CIDR | Consumes VPC subnet IPs |
| **Packet visibility on wire** | Outer node IPs only (unless decapped) | Native Pod IPs on wire | Native Pod IPs in VPC Flow Logs |

## Hands-on lab: Build container networking with `ip netns`

You can reproduce the exact mechanics of a bridge-based CNI plugin on any Linux host (or Linux VM) in under two minutes with zero container runtime installed.

### 1. Create two namespaces and a bridge

```bash
# Create two isolated network namespaces (simulating Pod A and Pod B)
sudo ip netns add pod-a
sudo ip netns add pod-b

# Create a Linux bridge in the root namespace and bring it up
sudo ip link add name cni0 type bridge
sudo ip addr add 10.244.1.1/24 dev cni0
sudo ip link set cni0 up
```

### 2. Wire `veth` pairs into each namespace

```bash
# Create veth pairs
sudo ip link add veth-a type veth peer name eth0-a
sudo ip link add veth-b type veth peer name eth0-b

# Move container ends into their namespaces and rename to eth0
sudo ip link set eth0-a netns pod-a
sudo ip netns exec pod-a ip link set eth0-a name eth0

sudo ip link set eth0-b netns pod-b
sudo ip netns exec pod-b ip link set eth0-b name eth0

# Attach host ends to the bridge and bring them up
sudo ip link set veth-a master cni0
sudo ip link set veth-b master cni0
sudo ip link set veth-a up
sudo ip link set veth-b up
```

### 3. Assign Pod IPs, routes, and verify

```bash
# Configure Pod A
sudo ip netns exec pod-a ip link set lo up
sudo ip netns exec pod-a ip addr add 10.244.1.10/24 dev eth0
sudo ip netns exec pod-a ip link set eth0 up
sudo ip netns exec pod-a ip route add default via 10.244.1.1

# Configure Pod B
sudo ip netns exec pod-b ip link set lo up
sudo ip netns exec pod-b ip addr add 10.244.1.11/24 dev eth0
sudo ip netns exec pod-b ip link set eth0 up
sudo ip netns exec pod-b ip route add default via 10.244.1.1

# Verify Pod A -> Pod B connectivity and inspect bridge MAC table
sudo ip netns exec pod-a ping -c 3 10.244.1.11
bridge fdb show br cni0
```

### 4. Enable outbound NAT and clean up

```bash
# Allow Pods to reach external networks via host forwarding + masquerade
sudo sysctl -w net.ipv4.ip_forward=1
sudo nft add table ip cni_nat
sudo nft 'add chain ip cni_nat postrouting { type nat hook postrouting priority srcnat; policy accept; }'
sudo nft add rule ip cni_nat postrouting ip saddr 10.244.1.0/24 oifname != "cni0" masquerade

# Clean up the lab when finished
sudo nft delete table ip cni_nat
sudo ip netns del pod-a
sudo ip netns del pod-b
sudo ip link del cni0
```

> [!WARNING]
> **Docker/iptables FORWARD policy trap:** If Docker is installed on your lab host, it sets the default `iptables` `FORWARD` chain policy to `DROP`. Packets traversing `cni0` between `pod-a` and `pod-b` may be dropped if `net.bridge.bridge-nf-call-iptables=1` is active unless you allow forwarding on `cni0`.

## Summary

Container networking is standard Linux networking: network namespaces isolate state, `veth` pairs bridge namespaces, and CNI plugins automate IPAM, routing, and cross-node transport via overlays or native BGP/cloud routing. Next, we layer Kubernetes Services, virtual IP load balancing, and eBPF dataplanes on top of Pod networking in [Kubernetes Services, kube-proxy, and Cilium eBPF](03-kubernetes-services-and-dataplanes.md).
