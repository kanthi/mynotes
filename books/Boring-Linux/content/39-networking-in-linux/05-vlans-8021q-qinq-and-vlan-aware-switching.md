---
title: 'Chapter 5: VLANs: 802.1Q, QinQ, and VLAN-Aware Switching'
---

# Chapter 5: VLANs: 802.1Q, QinQ, and VLAN-Aware Switching

Virtual LANs (IEEE 802.1Q) slice a single physical Ethernet link into up to 4,094 isolated Layer 2 broadcast domains by inserting a 4-byte tag inside the Ethernet header. Linux supports VLANs in two distinct, complementary ways:

1. **VLAN Subinterfaces (`type vlan`)** — the Linux equivalent of router subinterfaces (`GigabitEthernet0/1.100`), ideal for router-on-a-stick, server multi-VLAN NICs, and SVIs on top of bonds.
2. **VLAN-Aware Bridges (`vlan_filtering 1`)** — the Linux equivalent of a managed Layer 2 switch with access ports, trunk ports, PVIDs (native VLANs), and Switch Virtual Interfaces (SVIs), all in a single bridge object.

## IEEE 802.1Q and 802.1ad (QinQ) Frame Mechanics

```text
┌─────────┬─────────┬──────────────────────────────┬───────────┬─────────┬─────┐
│ Dst MAC │ Src MAC │       802.1Q Tag (4 B)       │ EtherType │ Payload │ FCS │
│  (6 B)  │  (6 B)  │ TPID=0x8100│PCP(3)│DEI│VID(12│   (2 B)   │ (≤1500) │(4 B)│
└─────────┴─────────┴──────────────────────────────┴───────────┴─────────┴─────┘
```

- **TPID:** `0x8100` for standard 802.1Q (C-VLAN); `0x88a8` for 802.1ad QinQ outer Service Tag (S-VLAN).
- **PCP (3 bits):** 802.1p Priority Code Point (`0`–`7`) for Layer 2 CoS/QoS marking.
- **DEI (1 bit):** Drop Eligible Indicator.
- **VID (12 bits):** VLAN ID `1`–`4094` (`0` = priority-tagged only; `4095` = reserved).

> [!NOTE]
> **MTU and VLAN tags:** Modern Linux NICs support 1504-byte (and 1508-byte QinQ) L2 frames automatically so that a VLAN subinterface keeps a full **1500-byte L3 MTU** while the parent carries the 4-byte 802.1Q header. Keep the parent physical interface `UP` (`ip link set enp1s0 up`), even if it has **no IP address** (for untagged native VLAN absence).

## Creating VLAN Subinterfaces (`ip link ... type vlan`)

Ensure the `8021q` kernel module is loaded (auto-loaded by `ip link add` on modern kernels):

```bash
# Parent interface must be UP (can have no IP if purely a trunk)
sudo ip link set dev enp1s0 up

# Create 802.1Q subinterfaces for VLAN 10 (Corp) and VLAN 20 (DMZ)
sudo ip link add link enp1s0 name enp1s0.10 type vlan id 10
sudo ip link add link enp1s0 name enp1s0.20 type vlan id 20

# Assign Layer 3 gateway IPs and bring them UP
sudo ip addr add 10.10.10.1/24 dev enp1s0.10
sudo ip addr add 10.10.20.1/24 dev enp1s0.20
sudo ip link set dev enp1s0.10 up
sudo ip link set dev enp1s0.20 up

# Inspect VLAN metadata (protocol, VID, reorder_hdr, QoS maps)
ip -d link show dev enp1s0.10
```

### QoS Priority Mapping (PCP) and Loose Binding

- **802.1p CoS mapping:** Map internal Linux `skb->priority` values to 802.1Q PCP bits (`0`–`7`) on egress and back on ingress:
  ```bash
  sudo ip link add link enp1s0 name enp1s0.30 type vlan id 30 \
    egress-qos-map 0:0 4:5 6:6 \
    ingress-qos-map 5:4 6:6
  ```
- **Loose binding (`loose_binding on`):** Normally, if parent `enp1s0` loses carrier, `enp1s0.10` goes down too. `loose_binding on` decouples the VLAN operstate from parent carrier (rarely needed except in specialized active-standby setups).

## Double-Tagging: IEEE 802.1ad (QinQ)

Service providers and multi-tenant data centers stack an outer **S-Tag (`0x88a8`)** around customer **C-Tags (`0x8100`)** so customer VLAN IDs can overlap cleanly across a shared transport network:

```bash
# 1. Create outer Service VLAN 100 (TPID 0x88a8)
sudo ip link add link enp1s0 name enp1s0.s100 type vlan protocol 802.1ad id 100
sudo ip link set enp1s0.s100 up

# 2. Stack inner Customer VLAN 200 (TPID 0x8100) on top of S-VLAN 100
sudo ip link add link enp1s0.s100 name enp1s0.s100.c200 type vlan protocol 802.1Q id 200
sudo ip link set enp1s0.s100.c200 up
sudo ip addr add 172.16.200.1/24 dev enp1s0.s100.c200
```

Verify double-tagged frames on the wire with `tcpdump`:

```bash
sudo tcpdump -e -nn -i enp1s0 'vlan 100 and vlan 200'
```

> [!WARNING]
> **Hardware VLAN offload gotcha with `tcpdump`:** Many NICs strip 802.1Q headers in hardware (`rx-vlan-offload: on`). Modern `tcpdump` reads the tag back from kernel ancillary metadata, **except** in BPF filter edge cases or older kernels. If a `tcpdump -i enp1s0 vlan 10` filter behaves unexpectedly, check `ethtool -k enp1s0 | grep vlan` and test without BPF filtering (`tcpdump -e -nn -i enp1s0`).

## VLAN-Aware Linux Bridges (The Modern Switch Model)

In the old Linux model, you had to create a separate bridge per VLAN (`br10`, `br20`, `br30`) and attach subinterfaces (`eth0.10`, `eth0.20`) to each — an ugly, error-prone mess.

Modern Linux kernels (3.9+, refined in 5.x/6.x) provide a **VLAN-aware bridge** (`vlan_filtering 1`), behaving identically to a Cisco Catalyst or Arista switch:

```bash
# Create one switch bridge with VLAN filtering enabled (default_pvid 1)
sudo ip link add name sw0 type bridge vlan_filtering 1 vlan_default_pvid 1
sudo ip link set sw0 up

# Attach trunk port (uplink) and access ports (port1, port2)
sudo ip link set eth0 master sw0 up
sudo ip link set eth1 master sw0 up
sudo ip link set eth2 master sw0 up

# Configure eth1 as ACCESS port in VLAN 10 (PVID 10 + untagged on egress)
sudo bridge vlan del vid 1 dev eth1
sudo bridge vlan add vid 10 pvid untagged dev eth1

# Configure eth2 as ACCESS port in VLAN 20
sudo bridge vlan del vid 1 dev eth2
sudo bridge vlan add vid 20 pvid untagged dev eth2

# Configure eth0 as TRUNK port carrying VLANs 10 and 20 tagged (and VLAN 99 native)
sudo bridge vlan del vid 1 dev eth0
sudo bridge vlan add vid 99 pvid untagged dev eth0
sudo bridge vlan add vid 10 dev eth0
sudo bridge vlan add vid 20 dev eth0

# Inspect the full switchport VLAN table (Cisco "show vlan brief" / "show int trunk")
bridge -c vlan show
```

### Adding SVIs (Switch Virtual Interfaces) to a VLAN-Aware Bridge

To route *between* VLAN 10 and VLAN 20 on the Linux switch itself (an L3 switch SVI), allow those VLANs onto the **bridge's own internal CPU port (`self`)** and create VLAN subinterfaces on `sw0`:

```bash
# Permit VLAN 10 and 20 to reach the Linux host CPU (bridge "self")
sudo bridge vlan add vid 10 dev sw0 self
sudo bridge vlan add vid 20 dev sw0 self

# Create SVI interfaces sw0.10 and sw0.20
sudo ip link add link sw0 name sw0.10 type vlan id 10
sudo ip link add link sw0 name sw0.20 type vlan id 20
sudo ip addr add 10.10.10.1/24 dev sw0.10
sudo ip addr add 10.10.20.1/24 dev sw0.20
sudo ip link set sw0.10 up
sudo ip link set sw0.20 up
```

## Persisting VLANs (Netplan and `systemd-networkd`)

### Netplan VLAN Subinterfaces

```yaml
# /etc/netplan/20-vlans.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0: {}
  vlans:
    vlan10:
      id: 10
      link: enp1s0
      addresses: [10.10.10.1/24]
    vlan20:
      id: 20
      link: enp1s0
      addresses: [10.10.20.1/24]
```

### Native `systemd-networkd` VLAN-Aware Switch Port

```ini
# /etc/systemd/network/20-sw0.netdev
[NetDev]
Name=sw0
Kind=bridge

[Bridge]
VLANFiltering=yes
DefaultPVID=none

# /etc/systemd/network/21-eth1-access.network
[Match]
Name=eth1

[Network]
Bridge=sw0

[BridgeVLAN]
PVID=10
EgressUntagged=10
```

## Hands-On Lab 5: VLAN-Aware Switch + Router-on-a-Stick

Build a complete 4-namespace topology: two end hosts (`pc10` in VLAN 10, `pc20` in VLAN 20), a VLAN-aware Linux switch (`sw`), and a Router-on-a-Stick (`rtr`) connected over an 802.1Q trunk.

```text
  [pc10: 10.10.10.10/24] ──(access VLAN 10)──┐
                                             ├──► [sw: br0 (vlan_filtering=1)]
  [pc20: 10.10.20.20/24] ──(access VLAN 20)──┘               │
                                                     (802.1Q trunk: 10, 20)
                                                             ▼
                                                [rtr: trunk.10 & trunk.20]
                                                 10.10.10.1   10.10.20.1
```

```bash
# 1. Create namespaces
for ns in pc10 pc20 sw rtr; do sudo ip netns add "$ns"; done

# 2. Wire cables
sudo ip link add eth0-pc10 type veth peer name sw-p1
sudo ip link add eth0-pc20 type veth peer name sw-p2
sudo ip link add sw-trunk  type veth peer name rtr-trunk

sudo ip link set eth0-pc10 netns pc10; sudo ip link set sw-p1 netns sw
sudo ip link set eth0-pc20 netns pc20; sudo ip link set sw-p2 netns sw
sudo ip link set sw-trunk  netns sw;   sudo ip link set rtr-trunk netns rtr

# 3. Configure VLAN-aware switch in namespace sw
sudo ip -n sw link add br0 type bridge vlan_filtering 1 vlan_default_pvid 0
sudo ip -n sw link set br0 up
for p in sw-p1 sw-p2 sw-trunk; do sudo ip -n sw link set "$p" master br0 up; done

# Access ports
sudo ip netns exec sw bridge vlan add vid 10 pvid untagged dev sw-p1
sudo ip netns exec sw bridge vlan add vid 20 pvid untagged dev sw-p2
# Trunk port carrying VLAN 10 and 20 tagged
sudo ip netns exec sw bridge vlan add vid 10 dev sw-trunk
sudo ip netns exec sw bridge vlan add vid 20 dev sw-trunk

# 4. Configure Router-on-a-Stick subinterfaces in namespace rtr
sudo ip -n rtr link set rtr-trunk up
sudo ip -n rtr link add link rtr-trunk name rtr-trunk.10 type vlan id 10
sudo ip -n rtr link add link rtr-trunk name rtr-trunk.20 type vlan id 20
sudo ip -n rtr addr add 10.10.10.1/24 dev rtr-trunk.10
sudo ip -n rtr addr add 10.10.20.1/24 dev rtr-trunk.20
sudo ip -n rtr link set rtr-trunk.10 up
sudo ip -n rtr link set rtr-trunk.20 up
sudo ip netns exec rtr sysctl -w net.ipv4.ip_forward=1

# 5. Configure end hosts (untagged Ethernet — they know nothing about VLANs!)
sudo ip -n pc10 link set eth0-pc10 up
sudo ip -n pc10 addr add 10.10.10.10/24 dev eth0-pc10
sudo ip -n pc10 route add default via 10.10.10.1

sudo ip -n pc20 link set eth0-pc20 up
sudo ip -n pc20 addr add 10.10.20.20/24 dev eth0-pc20
sudo ip -n pc20 route add default via 10.10.20.1

# 6. Verify inter-VLAN routing and observe 802.1Q tags on the trunk!
sudo ip netns exec sw bridge -c vlan show
sudo ip netns exec rtr timeout 3 tcpdump -l -e -nn -i rtr-trunk icmp &
sleep 0.5
sudo ip netns exec pc10 ping -c 2 10.10.20.20
wait

# 7. Cleanup
for ns in pc10 pc20 sw rtr; do sudo ip netns del "$ns"; done
```

**What you verified:** `tcpdump -e` on `rtr-trunk` displays `vlan 10` on the inbound ICMP Echo Request and `vlan 20` on the routed outbound frame, while `pc10` and `pc20` send and receive pure untagged frames.
