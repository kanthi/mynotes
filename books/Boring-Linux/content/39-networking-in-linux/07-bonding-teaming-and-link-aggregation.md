---
title: 'Chapter 7: Bonding, Teaming, and Link Aggregation'
---

# Chapter 7: Bonding, Teaming, and Link Aggregation

Link aggregation bundles multiple physical NICs into one logical interface (`bond0`) to provide **redundancy** (surviving cable, transceiver, NIC, or switch failures) and **aggregate throughput** across multiple flows. On Cisco/Arista switches you call it Port-Channel or LAG; on Juniper, `ae0`; on Linux, the kernel **`bonding`** driver handles it natively with full IEEE 802.3ad LACP support.

## All Seven Linux Bonding Modes

Linux's `bonding` module supports seven distinct operational modes. Choosing the wrong mode — or mismatching it with the upstream switch — is one of the most common datacenter misconfigurations.

| Mode | Name | Switch Support Required? | Load Balancing | Fault Tolerance | Best Use Case |
|---|---|---|---|---|---|
| `0` | `balance-rr` | Static LAG (no LACP) | Per-packet round-robin | Yes | Legacy only — **causes TCP out-of-order packets** on modern links |
| `1` | `active-backup` | **None** (any two independent switches!) | Single active link | Yes | Maximum simplicity, dual non-MLAG switches, management NICs |
| `2` | `balance-xor` | Static LAG / EtherChannel | Per-flow hash (`xmit_hash_policy`) | Yes | Static LAG where switch does not support LACP |
| `3` | `broadcast` | Special / dedicated | Transmits every frame on all slaves | Yes | Specialized ultra-low-latency financial / redundant telemetry |
| `4` | **`802.3ad`** | **LACP (or MLAG/vPC)** | Per-flow hash (`xmit_hash_policy`) | Yes (dynamic LACPDU health) | **Production datacenter standard** (servers, hypervisors, routers) |
| `5` | `balance-tlb` | None | Adaptive TX only; RX on 1 link | Yes | Asymmetric TX-heavy hosts without managed switch LAG |
| `6` | `balance-alb` | None | Adaptive TX + ARP-rewritten RX | Yes | Avoid with bridges/VMs (rewrites MACs via ARP replies) |

> [!IMPORTANT]
> **Single-flow bandwidth myth:** In `802.3ad` (LACP) and `balance-xor`, a **single TCP connection** hashes to **one** physical member link so packets stay in order. Two 10 GbE NICs in an LACP bond deliver **20 Gbps aggregate** across *many* flows, not 20 Gbps for a single `iperf3 -P 1` stream.

## Deep Dive: IEEE 802.3ad (LACP) Tuning

Getting LACP right between a Linux host and a datacenter switch (Cisco vPC, Arista MLAG, Juniper MC-LAG, Cumulus) requires matching four key knobs:

### 1. Transmit Hash Policy (`xmit_hash_policy`)

Controls how Linux selects an egress member link per packet:

| Policy | Fields Hashed | Recommendation |
|---|---|---|
| `layer2` (default) | Src/Dst MAC + VLAN | **Poor for routed traffic** (all traffic to one default gateway MAC hashes to a single link!) |
| `layer2+3` | MAC + Src/Dst IP | Good balance for L3 traffic without fragmenting fragmented UDP |
| **`layer3+4`** | **Src/Dst IP + TCP/UDP ports** | **Best distribution for servers, web/DB clusters, and routers** |
| `encap2+3` / `encap3+4` | Inner headers of VXLAN/GRE/IPIP tunnels | Essential on virtualization hosts carrying tunneled overlays |

### 2. LACP Rate (`lacp_rate`)

- `slow` (`0`, default): Exchanges LACPDUs every **30 seconds** (dead detection = 90 seconds).
- `fast` (`1`): Exchanges LACPDUs every **1 second** (dead detection = **3 seconds**). Always verify whether the upstream switch supports `lacp rate fast` (most modern ToR switches do).

### 3. Link Monitoring: `miimon` vs `arp_interval`

- **`miimon 100`** (standard): Checks NIC hardware carrier status every 100 ms. Pair with `updelay 200` / `downdelay 200` (multiples of `miimon`) to debounce flapping optics.
- **`arp_interval` + `arp_ip_target`**: Sends periodic ARP probes to verify end-to-end Layer 2 reachability (useful in `active-backup` when an intermediate media converter hides carrier loss; **incompatible with `802.3ad`**).

### 4. `min_links` and `ad_select`

- `min_links 1`: Minimum active links required before `bond0` reports carrier `UP` (set `min_links 2` if a 4-link bundle should fail over to another router once capacity drops below 50%).
- `ad_select`: Chooses which LACP aggregator wins if cables accidentally land on two different switches (`stable` default, `bandwidth`, `count`).

## Configuring Bonding with `iproute2`

```bash
# 1. Create an 802.3ad LACP bond with Layer3+4 hashing, fast LACP, and 100ms MII monitor
sudo ip link add bond0 type bond \
  mode 802.3ad \
  miimon 100 \
  lacp_rate fast \
  xmit_hash_policy layer3+4 \
  min_links 1

# 2. Slave NICs MUST be administratively DOWN before enslaving to a bond
sudo ip link set enp1s0f0 down
sudo ip link set enp1s0f1 down
sudo ip link set enp1s0f0 master bond0
sudo ip link set enp1s0f1 master bond0

# 3. Bring bond0 UP (kernel automatically brings slaves UP and negotiates LACP)
sudo ip link set bond0 up
sudo ip addr add 10.70.0.10/24 dev bond0

# 4. Inspect full LACP state (Aggregator ID, Partner System MAC, Oper Key, Actor/Partner state)
cat /proc/net/bonding/bond0
```

### Reading `/proc/net/bonding/bond0` Like a Pro

When LACP is healthy:
- **Both slaves** share the **exact same `Aggregator ID`** (e.g., `Aggregator ID: 1`) and the **exact same `Partner Mac Address`** (the switch / MLAG system ID).
- **Red flag:** If Slave 1 has `Aggregator ID: 1` and Slave 2 has `Aggregator ID: 2`, the two switch ports are **not** in the same Port-Channel / MLAG!
- **Red flag:** If `Partner Mac Address` is `00:00:00:00:00:00`, the switch is not sending LACPDUs (port is not configured for LACP active/passive, or STP/err-disabled).

## Stacking VLANs and Bridges on Top of a Bond

The canonical datacenter Linux server / hypervisor stack layers interfaces strictly from bottom to top:

```text
  [ Physical NICs: enp1s0f0 + enp1s0f1 ]
                     │
                     ▼  (Link Aggregation: LACP)
                 [ bond0 ]
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
   (Router / Server)     (Hypervisor / Switch)
    bond0.10 (Mgmt)      br0 (vlan_filtering=1, master of bond0)
    bond0.20 (Data)        ├── vm-tap0 (access VLAN 20)
                           └── br0.10 (SVI Mgmt IP)
```

> [!CAUTION]
> **Layering rule:** Never bond two VLAN subinterfaces (`eth0.10` + `eth1.10`) when you can bond the physical NICs first (`bond0`) and place VLANs (`bond0.10`) or a VLAN-aware bridge on top of `bond0`.

## `bonding` vs `teamd`

Years ago, Red Hat introduced `teamd` (a minimal kernel `team` driver paired with a userspace JSON daemon) as a modular alternative to `bonding`. Today, **`teamd` is deprecated** in RHEL 9/10 and removed from modern distributions. The kernel **`bonding`** driver has full Netlink support, zero userspace daemon crash risk, and universal Netplan / `systemd-networkd` / NetworkManager support. **Always choose kernel `bonding` for new deployments.**

## Persisting an LACP Bond (Netplan & `systemd-networkd`)

### Netplan (Ubuntu 26.04 LTS)

```yaml
# /etc/netplan/30-bond-lacp.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0f0: {}
    enp1s0f1: {}
  bonds:
    bond0:
      interfaces: [enp1s0f0, enp1s0f1]
      mtu: 9000
      parameters:
        mode: 802.3ad
        lacp-rate: fast
        mii-monitor-interval: 100
        transmit-hash-policy: layer3+4
  vlans:
    bond0.100:
      id: 100
      link: bond0
      addresses: [10.100.0.10/24]
```

### Native `systemd-networkd`

```ini
# /etc/systemd/network/10-bond0.netdev
[NetDev]
Name=bond0
Kind=bond
MTUBytes=9000

[Bond]
Mode=802.3ad
TransmitHashPolicy=layer3+4
MIIMonitorSec=100ms
LACPTransmitRate=fast

# /etc/systemd/network/20-bond-slaves.network
[Match]
Name=enp1s0f0 enp1s0f1

[Network]
Bond=bond0
```

## Hands-On Lab 7: Live 802.3ad LACP Negotiation and Failover Testing

Connect two Linux namespaces (`hostA` and `hostB`) with **two parallel virtual cables**, negotiate real IEEE 802.3ad LACP between both Linux bonding drivers, inspect `/proc/net/bonding/bond0`, and pull a cable during live traffic to verify hitless failover.

```bash
# 1. Create two namespaces wired with two parallel veth links
sudo ip netns add hostA
sudo ip netns add hostB

sudo ip link add a0 type veth peer name b0
sudo ip link add a1 type veth peer name b1
sudo ip link set a0 netns hostA; sudo ip link set a1 netns hostA
sudo ip link set b0 netns hostB; sudo ip link set b1 netns hostB

# 2. Configure 802.3ad LACP bond on BOTH hostA and hostB
for ns in hostA hostB; do
  sudo ip -n "$ns" link add bond0 type bond \
    mode 802.3ad miimon 100 lacp_rate fast xmit_hash_policy layer3+4
done

sudo ip -n hostA link set a0 down; sudo ip -n hostA link set a1 down
sudo ip -n hostA link set a0 master bond0
sudo ip -n hostA link set a1 master bond0
sudo ip -n hostA link set bond0 up
sudo ip -n hostA addr add 10.70.0.1/24 dev bond0

sudo ip -n hostB link set b0 down; sudo ip -n hostB link set b1 down
sudo ip -n hostB link set b0 master bond0
sudo ip -n hostB link set b1 master bond0
sudo ip -n hostB link set bond0 up
sudo ip -n hostB addr add 10.70.0.2/24 dev bond0

# 3. Wait 2 seconds for fast LACPDUs to converge, then inspect LACP Aggregator state!
sleep 2
sudo ip netns exec hostA cat /proc/net/bonding/bond0 | grep -E 'Bonding Mode|Slave Interface|MII Status|Aggregator ID|Partner Mac'

# 4. Start a continuous ping and sever link a0 (simulating a fiber cut!)
sudo ip netns exec hostA ping -i 0.2 -c 15 10.70.0.2 &
PING_PID=$!
sleep 1
echo "--- Cutting link a0 ---"
sudo ip -n hostA link set a0 down
wait "$PING_PID"

# 5. Inspect degraded bond state (a0 DOWN, a1 still UP and carrying traffic)
sudo ip netns exec hostA cat /proc/net/bonding/bond0 | grep -E 'Slave Interface|MII Status|Number of ports'

# 6. Restore link a0 and verify LACP re-aggregates automatically
sudo ip -n hostA link set a0 up
sleep 2
sudo ip netns exec hostA cat /proc/net/bonding/bond0 | grep -E 'Slave Interface|MII Status|Number of ports'

# 7. Cleanup
sudo ip netns del hostA
sudo ip netns del hostB
```

**What you verified:** Both namespaces exchanged real `0x8809` LACPDUs, merged both links into `Number of ports: 2`, survived a simulated fiber cut on `a0` with minimal/zero packet loss, and seamlessly re-joined the LAG upon link recovery.
