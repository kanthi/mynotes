---
title: 'Chapter 14: Virtual Networking, Overlays, Containers, and tc/eBPF'
---

# Chapter 14: Virtual Networking, Overlays, Containers, and tc/eBPF

Cloud platforms, KVM/QEMU hypervisors, Docker/Podman, and Kubernetes CNI plugins do not invent magical networking primitives — they compose **standard Linux kernel virtual network devices**: `netns`, `veth`, `macvlan`, `ipvlan`, `TUN`/`TAP`, **VXLAN**, **Geneve**, **Open vSwitch (OVS)**, Traffic Control (**`tc`**), and programmable **eBPF/XDP**. Understanding these building blocks demystifies every container and cloud virtual network instantly.

## Virtual Network Device Taxonomy

| Device Type | Layer | Kernel Mechanism | Primary Use Case |
|---|---|---|---|
| **`veth`** | L2 | Bi-directional virtual Ethernet wire (always created in pairs) | Connecting containers/namespaces to bridges, OVS, or host stack |
| **`bridge`** | L2 | Software Ethernet switch (FDB + STP + VLAN filtering) | KVM hypervisors, `docker0`, Linux switches |
| **`macvlan`** | L2 | Virtual NIC sharing a physical parent NIC, with its **own unique MAC** | Lightweight container/VM direct-attach to physical LAN (no bridge overhead) |
| **`ipvlan`** | L2/L3 | Shares parent NIC's **same MAC**, demuxes by IP (L2, L3, or L3S mode) | Strict switchport-security environments (1 MAC/port) & cloud VPCs |
| **`TAP`** | L2 | Exposes Ethernet frames to a userspace file descriptor (`/dev/net/tun`) | KVM/QEMU virtual NICs (`vhost-net` kernel fast-path), Layer 2 VPNs |
| **`TUN`** | L3 | Exposes raw IP packets to a userspace process | OpenVPN, Tailscale/WireGuard userspace, sshuttle |
| **`vxlan`** | L2-over-L3 | Encapsulates Ethernet frames in UDP (port 4789) with a 24-bit VNI (16M segments) | Multi-tenant datacenter fabrics, BGP EVPN, Flannel/Cilium overlays |
| **`geneve`** | L2-over-L3 | Extensible TLV-based UDP tunnel (port 6081) | OVN / Open vSwitch cloud SDN fabrics |

### `macvlan` Gotcha: Host-to-Macvlan Isolation

By design, a `macvlan` child interface in `bridge` mode can reach the physical network and sibling `macvlan` interfaces on the same parent, **but cannot talk directly to the parent interface's own IP on the host!** To allow host↔container communication over `macvlan`, create a second `macvlan` interface on the host itself and route the container subnet through it.

## VXLAN Overlays: Extending Layer 2 Over Layer 3

VLANs stop at 4,094 IDs and require Layer 2 stretching (with Spanning Tree headaches). **VXLAN** (RFC 7348) encapsulates full Layer 2 Ethernet frames inside **Layer 3 UDP/4789 packets**, giving you **16,777,216 isolated segments (VNIs)** routed over any ECMP Layer 3 spine-leaf fabric:

```text
┌─────────┬──────────┬─────────┬───────────────┬─────────────────────────────┐
│ Outer   │ Outer IP │ Outer   │ VXLAN Header  │ Original Inner L2 Frame     │
│ Eth(14) │ (20 B)   │ UDP(8)  │ (8 B: 24b VNI)│ (Inner Eth + IP + Payload)  │
└─────────┴──────────┴─────────┴───────────────┴─────────────────────────────┘
                Total encapsulation overhead = 50 bytes (IPv4) / 70 bytes (IPv6)
```

Three ways Linux VXLAN VTEPs (VXLAN Tunnel Endpoints) discover remote MACs:
1. **Multicast underlay:** VTEPs join a multicast group (`group 239.1.1.1`) for BUM (Broadcast, Unknown-unicast, Multicast) flooding.
2. **Head-End Replication (HER) static FDB:** Explicitly list remote VTEP IPs (`bridge fdb append 00:00:00:00:00:00 dev vxlan100 dst 198.51.100.2`) — works on unicast-only cloud networks.
3. **BGP EVPN Control Plane (FRRouting):** Eliminates BUM flooding altogether! FRR exchanges Type-2 (MAC/IP) and Type-3 (IMET) EVPN routes over BGP and programs `bridge fdb` + `ip neigh` entries directly into the kernel via Netlink (`nolearning`).

## Open vSwitch (OVS) and Container Networking Under the Hood

### Open vSwitch (`ovs-vswitchd` + `openvswitch` kernel datapath)

OVS provides programmable OpenFlow switching, native VXLAN/Geneve tunnels, QoS policing, sFlow/IPFIX telemetry, and serves as the switching layer for OpenStack and OVN-Kubernetes:

```bash
# Create an OVS bridge, attach a port with VLAN tag 100, and add a VXLAN tunnel port
sudo ovs-vsctl add-br ovs-br0
sudo ovs-vsctl add-port ovs-br0 eth1 tag=100
sudo ovs-vsctl add-port ovs-br0 vxlan0 -- set interface vxlan0 type=vxlan \
  options:remote_ip=198.51.100.2 options:key=100

# Inspect OVS topology and live kernel datapath flow cache
sudo ovs-vsctl show
sudo ovs-dpctl dump-flows
```

### How Container Networking Really Works (Docker, Podman, Kubernetes CNI)

Strip away the YAML and every container network reduces to three kernel operations:
1. Runtime creates a new **network namespace** for the container/pod.
2. Runtime creates a **`veth` pair**: one end becomes `eth0` inside the container namespace; the host end attaches either to a Linux bridge (`docker0` / `cni0` + `nftables` MASQUERADE), a pure L3 routed `/32` host link (**Calico**), or an **eBPF program** attached to `tc`/`XDP` (**Cilium**) bypassing bridges and `iptables` entirely!
3. Rootless containers (**Podman** without root) use **`pasta`** (or legacy `slirp4netns`) to forward sockets across user namespaces without requiring `CAP_NET_ADMIN` on the host.

## Traffic Control (`tc`): Shaping, Queueing, and Bufferbloat Elimination

Linux Traffic Control (`tc`) manages egress (and ingress) packet scheduling through **Qdiscs** (Queueing Disciplines), **Classes**, and **Filters**:

- **`fq_codel`** (default on modern Linux): Fair Queueing + Controlled Delay — automatically isolates interactive flows (SSH, DNS, VoIP) from bulk downloads and eliminates bufferbloat.
- **`cake`** (Common Applications Kept Enhanced): The ultimate single-command WAN shaper — combines deficit-round-robin flow fairness, per-host NAT-aware fairness, DiffServ priority tiers, and overhead framing compensation:
  ```bash
  # Shape a 100 Mbps asymmetrical WAN uplink cleanly with CAKE (zero bufferbloat!)
  sudo tc qdisc replace dev wan0 root cake bandwidth 100mbit docsis nat wash besteffort
  ```
- **`netem`** (Network Emulator): Inject realistic latency, jitter, packet loss, duplication, or corruption into any lab interface to test application resilience!
  ```bash
  # Inject 40ms +/- 5ms jitter and 1% random packet loss on eth0
  sudo tc qdisc replace dev eth0 root netem delay 40ms 5ms distribution normal loss 1%
  # Remove netem cleanly
  sudo tc qdisc del dev eth0 root
  ```
- **Inspect live qdisc stats (drops, overlimits, backlog):**
  ```bash
  tc -s -d qdisc show dev wan0
  ```

## eBPF and XDP: Programmable Kernel Data Plane

**XDP (eXpress Data Path)** executes verified eBPF bytecode directly inside the NIC driver's RX ring loop — **before** the kernel even allocates `struct sk_buff`!

| XDP Return Verdict | Action | Performance Context |
|---|---|---|
| `XDP_DROP` | Drop frame immediately in driver ring | 20–40+ Mpps per core DDoS mitigation |
| `XDP_TX` | Bounce modified frame right back out same NIC | Ultra-fast L4 load balancers (Katran) |
| `XDP_REDIRECT` | Redirect frame to another NIC, CPU, or `AF_XDP` userspace socket | Wire-speed routing / DPDK replacement |
| `XDP_PASS` | Hand frame upward to normal Linux stack | Selective inspection & filtering |

Inspect attached XDP and `tc` eBPF programs on any host:

```bash
ip -d link show | grep -E 'xdp|prog/xdp'
tc filter show dev eth0 ingress
sudo bpftool net list
```

## Hands-On Lab 14: Multi-Host VXLAN L2 Overlay Over a Routed L3 Underlay + `tc netem`

Extend a single Layer 2 broadcast domain (`10.200.0.0/24`) between two hosts (`vm1` and `vm2`) living behind two separate VTEPs (`vtep1` and `vtep2`) across an L3 router (`spine`) using **VXLAN VNI 5000** with Head-End Replication, and verify with `tc netem` latency injection!

```bash
# 1. Create namespaces: vm1, vtep1, spine, vtep2, vm2
for ns in vm1 vtep1 spine vtep2 vm2; do sudo ip netns add "$ns"; done

# 2. Wire L2 tenant links + L3 spine underlay
sudo ip link add vm1-eth0   type veth peer name vtep1-vm1
sudo ip link add vtep1-up   type veth peer name spine-1
sudo ip link add spine-2    type veth peer name vtep2-up
sudo ip link add vtep2-vm2  type veth peer name vm2-eth0

sudo ip link set vm1-eth0  netns vm1;   sudo ip link set vtep1-vm1 netns vtep1
sudo ip link set vtep1-up  netns vtep1; sudo ip link set spine-1   netns spine
sudo ip link set spine-2   netns spine; sudo ip link set vtep2-up  netns vtep2
sudo ip link set vtep2-vm2 netns vtep2; sudo ip link set vm2-eth0  netns vm2

# 3. Configure L3 Underlay (MTU 1600 on underlay so inner VXLAN keeps full 1500 MTU!)
sudo ip -n vtep1 link set vtep1-up mtu 1600 up
sudo ip -n vtep1 addr add 172.16.1.2/30 dev vtep1-up
sudo ip -n vtep1 route add default via 172.16.1.1

sudo ip -n spine link set spine-1 mtu 1600 up; sudo ip -n spine addr add 172.16.1.1/30 dev spine-1
sudo ip -n spine link set spine-2 mtu 1600 up; sudo ip -n spine addr add 172.16.2.1/30 dev spine-2
sudo ip netns exec spine sysctl -w net.ipv4.ip_forward=1

sudo ip -n vtep2 link set vtep2-up mtu 1600 up
sudo ip -n vtep2 addr add 172.16.2.2/30 dev vtep2-up
sudo ip -n vtep2 route add default via 172.16.2.1

# 4. Create VXLAN VNI 5000 + Tenant Bridge on vtep1 and vtep2 (unicast HER FDB)
sudo ip -n vtep1 link add vxlan5000 type vxlan id 5000 local 172.16.1.2 dstport 4789 dev vtep1-up
sudo ip -n vtep1 link add br5000 type bridge
sudo ip -n vtep1 link set vtep1-vm1 master br5000 up
sudo ip -n vtep1 link set vxlan5000 master br5000 up
sudo ip -n vtep1 link set br5000 up
sudo ip netns exec vtep1 bridge fdb append 00:00:00:00:00:00 dev vxlan5000 dst 172.16.2.2

sudo ip -n vtep2 link add vxlan5000 type vxlan id 5000 local 172.16.2.2 dstport 4789 dev vtep2-up
sudo ip -n vtep2 link add br5000 type bridge
sudo ip -n vtep2 link set vtep2-vm2 master br5000 up
sudo ip -n vtep2 link set vxlan5000 master br5000 up
sudo ip -n vtep2 link set br5000 up
sudo ip netns exec vtep2 bridge fdb append 00:00:00:00:00:00 dev vxlan5000 dst 172.16.1.2

# 5. Assign SAME L2 subnet 10.200.0.0/24 to vm1 and vm2 (NO default gateway needed!)
sudo ip -n vm1 link set vm1-eth0 up; sudo ip -n vm1 addr add 10.200.0.11/24 dev vm1-eth0
sudo ip -n vm2 link set vm2-eth0 up; sudo ip -n vm2 addr add 10.200.0.22/24 dev vm2-eth0

# 6. Inject 25ms WAN latency on spine using tc netem, then ping across the VXLAN overlay!
sudo ip netns exec spine tc qdisc replace dev spine-2 root netem delay 25ms
sudo ip netns exec spine timeout 3 tcpdump -l -nn -i spine-1 'udp port 4789' &
sleep 0.5
sudo ip netns exec vm1 ping -c 3 10.200.0.22
wait

# 7. Notice vm1 resolved vm2's direct MAC over ARP across the L3 spine!
sudo ip -n vm1 neigh show
sudo ip netns exec vtep1 bridge fdb show dev vxlan5000

# 8. Cleanup
for ns in vm1 vtep1 spine vtep2 vm2; do sudo ip netns del "$ns"; done
```

**What you verified:** `vm1` and `vm2` resolved each other's MAC addresses via Layer 2 ARP over a routed Layer 3 spine inside UDP/4789 VXLAN VNI 5000 packets, and `tc netem` added a realistic 25 ms one-way RTT visible directly in `ping`.
