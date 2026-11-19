---
title: 'Chapter 6: Linux Bridges and Software Switching'
---

# Chapter 6: Linux Bridges and Software Switching

The Linux kernel `bridge` module (`net/bridge/`) is a full-featured, production-grade IEEE 802.1D/802.1Q Layer 2 switch. It powers KVM/QEMU hypervisors, Docker/container bridges, OpenWrt routers, and white-box switches running `switchdev` hardware offload.

Understanding the Linux bridge means understanding how it learns MAC addresses into its Forwarding Database (FDB), how it floods unknown unicast/broadcast/multicast, how STP/RSTP prevents loops, how multicast snooping avoids flooding every port, and how bridge frames interact with Netfilter.

## Linux Bridge Architecture

When you enslave a network interface (`eth1`) to a bridge (`br0`) via `ip link set eth1 master br0`:
1. The kernel attaches a receive hook (`br_handle_frame`) to `eth1` and puts `eth1` into **promiscuous mode**.
2. Any IP address previously configured on `eth1` stops working properly — **never assign Layer 3 IPs to enslaved bridge ports**! Layer 3 IPs belong only on the bridge device itself (`br0`) or its VLAN SVIs (`br0.10`).
3. The bridge device `br0` acts as **both** the virtual switching backplane and the internal host port (`self`) connecting the Linux L3 stack to that switch.

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                   LINUX BRIDGE INTERNAL ARCHITECTURE                    │
│                                                                         │
│                       ┌───────────────────────────┐                     │
│                       │  Linux L3 Stack (IP/Route)│                     │
│                       └─────────────▲─────────────┘                     │
│                                     │                                   │
│                             [ br0 (self/CPU) ]                          │
│         ┌───────────────────────────┴───────────────────────────┐       │
│         │              Bridge Forwarding Engine                 │       │
│         │   • FDB (MAC + VID → Port)   • STP Port States        │       │
│         │   • MDB (IGMP/MLD Snooping)  • Hairpin / Isolated     │       │
│         └───────┬───────────────────┬───────────────────┬───────┘       │
│                 │                   │                   │               │
│            [ eth1 ]            [ eth2 ]            [ vnet0 (VM) ]       │
│         (Bridge Port)       (Bridge Port)         (Bridge Port)         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Managing the Forwarding Database (FDB)

Every frame entering a bridge port causes the kernel to learn `{Source MAC, VLAN ID} → Ingress Port` in the FDB (aging out after `ageing_time`, default 300 seconds):

```bash
# Create bridge with explicit aging time (300s) and forward delay
sudo ip link add name br0 type bridge ageing_time 30000 stp_state 0
sudo ip link set br0 up

# Inspect the MAC address table (Cisco "show mac address-table")
bridge -c fdb show br br0

# Add a static FDB entry (disable aging/movement for a pinned MAC)
sudo bridge fdb add 52:54:00:aa:bb:cc dev eth1 vlan 10 master static

# Disable MAC learning and unknown-unicast flooding on a hardened VM port
sudo bridge link set dev vnet0 learning off flood off
```

Understanding FDB flags in `bridge fdb show`:
- **`master`**: Entry lives in the software bridge's FDB table.
- **`self`**: Entry lives in the NIC hardware/driver table (critical for VXLAN and `switchdev`).
- **`permanent`**: Local MAC address of a bridge port itself (destined for local CPU).
- **`static`**: Manually configured, will not age out, but can be overwritten if moved unless `sticky` is set.
- **`offload`**: Programmed directly into switch ASIC silicon via `switchdev`.

## Spanning Tree Protocol (STP) and Loop Prevention

If you connect two Linux bridges with redundant links without a loop-prevention protocol, a single ARP broadcast creates an instant broadcast storm that pegs CPUs at 100%.

### Kernel 802.1D STP vs Userspace RSTP (`mstpd`)

The Linux kernel natively implements **legacy IEEE 802.1D STP** (slow 30–50s convergence):

```bash
# Enable kernel 802.1D STP on br0 with priority 4096 (lower = preferred Root Bridge)
sudo ip link set br0 type bridge stp_state 1 priority 4096

# Adjust per-port path cost and priority (lower cost = preferred uplink)
sudo bridge link set dev eth0 cost 4 priority 32

# Enable BPDU Guard (puts port in disabled state if unexpected BPDU arrives on an edge port)
sudo bridge link set dev eth1 guard on

# Enable Root Guard (prevents a downstream switch from claiming Root Bridge)
sudo bridge link set dev eth2 root_block on

# Inspect bridge port STP states (forwarding, blocking, listening, learning, disabled)
bridge -d link show
```

> [!TIP]
> For **Rapid STP (802.1w RSTP)** or **Multiple STP (802.1s MSTP)** with sub-second convergence on production Linux switches, run the userspace daemon **`mstpd`**. When `mstpd` is active, the kernel delegates BPDU state transitions to userspace while still switching data frames in kernel space.

## Multicast Snooping, Port Isolation, and Hairpin Mode

### IGMP / MLD Multicast Snooping

By default (`mcast_snooping 1`), the Linux bridge snoops IGMP (IPv4) and MLD (IPv6) membership reports so multicast streams are forwarded **only** to ports with interested listeners instead of flooding every port like broadcasts:

```bash
# Inspect the Multicast Database (MDB)
bridge -c mdb show

# Tune snooping or enable an IGMP querier when no external L3 multicast router exists
sudo ip link set br0 type bridge mcast_snooping 1 mcast_querier 1
```

### Port Isolation (Private VLAN / PVLAN)

To allow multiple tenant ports (e.g., VMs or hotel/Wi-Fi ports) to talk **only** to the uplink gateway and **never** directly to each other at Layer 2:

```bash
# Mark tenant ports isolated (isolated ports cannot forward to other isolated ports)
sudo bridge link set dev vm1 isolated on
sudo bridge link set dev vm2 isolated on
# Leave uplink port isolated off (promiscuous/uplink port can reach everyone)
sudo bridge link set dev eth0 isolated off
```

### Hairpin Mode (Reflective Relay)

Normally, a switch **never** forwards a frame back out the exact same port it arrived on (split-horizon rule). In virtualization or container scenarios where traffic must bounce off an external edge switch or single-port NAT hairpins back into the same bridge port, enable hairpin mode:

```bash
sudo bridge link set dev veth0 hairpin on
```

## Bridge Interaction with Netfilter (`br_netfilter`)

A notorious Linux gotcha occurs when Docker or Kubernetes loads the `br_netfilter` kernel module:

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
```

- When **`bridge-nf-call-iptables = 0`** (pure L2 switch): Bridged frames traverse **only** the `bridge` Netfilter family (Layer 2 `nftables` `table bridge ...`), bypassing Layer 3 `inet`/`ip` firewall chains completely.
- When **`bridge-nf-call-iptables = 1`**: Even pure Layer 2 bridged frames are pushed up through Layer 3 `iptables`/`nftables` `FORWARD` chains! On pure L2 Linux switches or KVM hypervisors that shouldn't firewall bridged guest frames, disable it cleanly:
  ```ini
  # /etc/sysctl.d/50-bridge-netfilter.conf
  net.bridge.bridge-nf-call-iptables = 0
  net.bridge.bridge-nf-call-ip6tables = 0
  net.bridge.bridge-nf-call-arptables = 0
  ```

For true Layer 2 firewalling (MAC filtering, ARP anti-spoofing, EtherType filtering), use **`nftables` `table bridge`**:

```nft
table bridge l2filter {
    chain forward {
        type filter hook forward priority filter; policy accept;
        # Block rogue DHCP servers on tenant access port eth1
        iifname "eth1" udp sport 67 drop
    }
}
```

## Hands-On Lab 6: Redundant Two-Switch STP Loop, FDB Inspection, and Port Isolation

Build two Linux bridges (`sw1` and `sw2`) wired with **two parallel redundant cables** (a physical Layer 2 loop!), watch STP block one link to prevent a broadcast storm, verify port isolation, and test automatic failover when the active link goes down.

```bash
# 1. Create namespaces: sw1, sw2, host1, host2
for ns in sw1 sw2 host1 host2; do sudo ip netns add "$ns"; done

# 2. Wire dual redundant links between sw1 and sw2 (Link A + Link B) + two hosts on sw1
sudo ip link add sw1-a type veth peer name sw2-a
sudo ip link add sw1-b type veth peer name sw2-b
sudo ip link add h1-eth0 type veth peer name sw1-h1
sudo ip link add h2-eth0 type veth peer name sw1-h2

sudo ip link set sw1-a netns sw1; sudo ip link set sw1-b netns sw1
sudo ip link set sw2-a netns sw2; sudo ip link set sw2-b netns sw2
sudo ip link set sw1-h1 netns sw1; sudo ip link set sw1-h2 netns sw1
sudo ip link set h1-eth0 netns host1; sudo ip link set h2-eth0 netns host2

# 3. Create STP-enabled bridges (sw1 = Root Bridge priority 4096, sw2 = 32768)
#    Use short forward_delay (200 centiseconds = 2s) so lab converges quickly!
sudo ip -n sw1 link add br0 type bridge stp_state 1 priority 4096 forward_delay 200
sudo ip -n sw2 link add br0 type bridge stp_state 1 priority 32768 forward_delay 200
sudo ip -n sw1 link set br0 up
sudo ip -n sw2 link set br0 up

for p in sw1-a sw1-b sw1-h1 sw1-h2; do sudo ip -n sw1 link set "$p" master br0 up; done
for p in sw2-a sw2-b; do sudo ip -n sw2 link set "$p" master br0 up; done

# 4. Wait ~5s for STP election and inspect port states on sw2
#    One link on sw2 (sw2-a or sw2-b) will be 'forwarding', the other 'blocking'!
sleep 5
sudo ip netns exec sw2 bridge -d link show | grep -E 'sw2-|state'

# 5. Give host1 and host2 IPs and test communication + inspect FDB
sudo ip -n host1 link set h1-eth0 up
sudo ip -n host1 addr add 10.60.0.1/24 dev h1-eth0
sudo ip -n host2 link set h2-eth0 up
sudo ip -n host2 addr add 10.60.0.2/24 dev h2-eth0

sudo ip netns exec host1 ping -c 2 10.60.0.2
sudo ip netns exec sw1 bridge -c fdb show br br0

# 6. Enable Port Isolation on sw1-h1 and sw1-h2 — direct host-to-host L2 traffic is blocked!
sudo ip netns exec sw1 bridge link set dev sw1-h1 isolated on
sudo ip netns exec sw1 bridge link set dev sw1-h2 isolated on
sudo ip netns exec host1 ping -c 2 -W 1 10.60.0.2 || echo "Blocked by Bridge Port Isolation!"

# 7. Cleanup
for ns in sw1 sw2 host1 host2; do sudo ip netns del "$ns"; done
```

**What you verified:** STP automatically elected `sw1` as Root Bridge and placed one of `sw2`'s redundant links into `blocking` state to stop a Layer 2 loop, `bridge fdb` learned both host MACs dynamically, and `isolated on` enforced Private VLAN host isolation in a single command.
