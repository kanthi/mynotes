---
title: 'Optics, Switch ASICs, and Lossless AI/ML Fabrics'
---

# Optics, Switch ASICs, and Lossless AI/ML Fabrics

Routing tables and EVPN control planes mean nothing if a dirty fiber connector drops frames with CRC errors, a shallow switch buffer tail-drops bursts during storage incast, or a GPU cluster stalls waiting on retransmitted RDMA packets. This chapter connects Layer 1 optics and switch silicon architecture to modern **lossless Ethernet fabrics for AI/ML clusters**.

## Learning goals

By the end of this chapter you can:

- Choose between optical form factors (**SFP+, QSFP28, QSFP-DD, OSFP**), fiber types (**SMF vs MMF**), and copper/active cables (**DAC vs AOC**)
- Explain why **RS-FEC** is mandatory at 100G/400G/800G and how FEC mismatches cause silent link failures
- Diagnose failing optics using **DOM/DDM** optical power budgets (`ethtool -m`, `show interfaces transceiver`)
- Contrast **shared-buffer vs VOQ** switch ASICs and **cut-through vs store-and-forward** switching
- Explain how **RoCEv2, PFC, ECN/DCQCN**, and **rail-optimized Clos topologies** power lossless AI/ML Ethernet fabrics

## Optical form factors and cabling

Data-center link speeds scale by increasing per-lane electrical/optical SerDes rates (10G → 25G → 50G PAM4 → 100G PAM4 → 200G PAM4) and bundling multiple lanes into one pluggable module.

| Form Factor | Lanes | Typical Speeds | Power / Thermal Profile | Common Role |
|---|---|---|---|---|
| **SFP+ / SFP28** | 1 | 10G / 25G | ~1–1.5 W | Server NICs, access uplinks |
| **QSFP28** | 4 × 25G | 100G (or 4×25G breakout) | ~3.5–4.5 W | Spine-leaf links, server uplinks |
| **QSFP-DD** | 8 × 50G or 8 × 100G | 400G / 800G | ~12–15 W | High-density 400G/800G fabrics; backward-compatible with QSFP28 |
| **OSFP** | 8 × 50G or 8 × 100G | 400G / 800G / 1.6T | ~15–25 W (integrated heatsink) | AI/ML spine/leaf switches and 800G/1.6T optics |

### DAC, AOC, MMF, and SMF

| Medium | Typical Reach | Pros | Cons |
|---|---|---|---|
| **DAC (Direct Attach Copper)** | 1–3 m (passive), up to 5 m (active) | Cheapest, lowest latency, lowest power | Thick/stiff bend radius, short reach (same rack only) |
| **AOC (Active Optical Cable)** | 3–30 m | Thin, light, factory-terminated (no dirty connectors) | Fixed length; if one end fails, replace whole cable |
| **MMF (Multi-Mode Fiber — OM3/OM4/OM5, aqua/violet)** | 70–150 m (SR/SR4/SR8, 850 nm) | Lower optic cost for short intra-row/intra-hall runs | Limited distance; modal dispersion at 400G/800G |
| **SMF (Single-Mode Fiber — OS2, yellow)** | 500 m (DR4), 2 km (FR4), 10 km (LR4), 40 km+ (ER4/ZR) | Future-proof cabling plant across generations (10G → 800G on same fiber) | Higher optic cost (tighter laser alignment; clean connectors strictly) |

> [!TIP]
> **Cabling rule of thumb:** Use **DAC** inside a single rack (ToR switch to servers), **SMF (DR4/FR4/LR4)** for leaf-spine and cross-row trunks so you never have to re-pull fiber when upgrading from 100G to 400G/800G.

## Forward Error Correction (FEC)

At 100G (4×25G) and especially 400G/800G (PAM4 modulation, 4 voltage levels per symbol instead of 2), raw bit error rates (pre-FEC BER) are too high for error-free Ethernet without math. **Reed-Solomon FEC (RS-FEC, Clause 91 / Clause 108)** adds parity symbols to every codeword so the receiver corrects bit errors on the fly at the cost of ~100–250 ns of latency.

- **Critical operational trap:** Both ends of a link must agree on the **exact same FEC mode** (`RS-FEC`, `FC-FEC / Firecode`, or `None`). A FEC mismatch leaves the link permanently `down` or flapping with massive symbol errors, even when optical light levels look perfect.
- **Pre-FEC BER vs Post-FEC Uncorrectable Errors:** A healthy 100G/400G link routinely shows non-zero **pre-FEC corrected codewords** (normal!). Alarm immediately on **post-FEC uncorrectable codewords** — those become dropped frames and CRC errors.

## DOM / DDM optical power diagnostics

Pluggable transceivers expose **Digital Optical Monitoring (DOM / DDM)** telemetry: temperature, supply voltage, laser bias current, **TX optical power (dBm)**, and **RX optical power (dBm)**.

```bash
# Linux server NIC: inspect transceiver DOM/DDM readings
sudo ethtool -m eth0

# Arista EOS / Cisco NX-OS: inspect optical power and alarm thresholds
# show interfaces Ethernet1 transceiver detail
```

Example diagnostic reading:

```text
Laser output power (TX) : -1.85 dBm   (normal: -4.0 to +2.0 dBm)
Receiver signal power (RX): -14.20 dBm  (LOW WARN: -10.0 dBm, LOW ALARM: -13.0 dBm)
```

- **TX normal, RX very low (`-14 dBm` or `-40 dBm` / no light):** Dirty fiber endface, pinched patch cord, wrong polarity (TX wired to TX instead of RX), or bad patch panel splice. Clean both ends with a one-click fiber cleaner before swapping optics.
- **TX low or high laser bias current:** Dying laser diode inside the transmitting optic — replace the remote transceiver.
- **RX too high (receiver overload):** Connecting a long-reach LR4/ER4 optic over a 2-meter patch cable without an optical attenuator can saturate or permanently damage the photodiode!

## Switch ASIC fundamentals

High-speed data-center switches forward packets in fixed-function **ASICs** (Broadcom Tomahawk/Jericho, NVIDIA Spectrum, Cisco Silicon One) at tens of terabits per second. Two architectural choices shape how a switch behaves under congestion:

### Cut-through vs Store-and-forward

| Mode | How It Forwards | Latency | Tradeoff |
|---|---|---|---|
| **Cut-through** | Starts transmitting frame out the egress port as soon as the destination MAC/IP header is parsed (~300–900 ns) | Lowest (ideal for HPC, trading, AI) | Cannot drop corrupted CRC frames before forwarding (propagates stomped frames); only works when ingress speed ≥ egress speed |
| **Store-and-forward** | Buffers the entire frame, verifies the 32-bit FCS/CRC, then forwards | Slightly higher (depends on frame size) | Drops bad frames immediately; required when stepping down speeds (e.g., 400G → 100G) |

### Shallow shared buffers vs Deep VOQ buffers

- **Shared-memory shallow-buffer ASICs (e.g., Tomahawk / Spectrum class, ~64–160 MB on-chip SRAM):** Ultra-low latency, high port density, designed for uniform east-west Clos fabrics where traffic stays within a few microseconds of congestion. Vulnerable to **incast** (many senders simultaneously blasting one receiver, exhausting the egress port's share of the buffer).
- **Deep-buffer VOQ (Virtual Output Queue) ASICs (e.g., Jericho / Qumran class, gigabytes of external HBM/GDDR):** Buffers packets on the **ingress** pipeline in per-egress-port queues so one congested egress port cannot cause **Head-of-Line (HoL) blocking** for other ports. Essential for WAN peering edges, storage fabrics, and mixed-speed fan-in (100G → 10G).

## Lossless AI/ML Ethernet fabrics

Distributed AI training (collective operations like `AllReduce` and `AllToAll` across thousands of GPUs) generates **synchronized, elephant-flow bursts** over **RDMA over Converged Ethernet v2 (RoCEv2)**. Traditional TCP handles packet loss gracefully with retransmissions; classic RDMA implementations historically suffered catastrophic throughput collapse (Go-Back-N retransmissions) on even 0.1% packet loss.

Modern Ethernet AI fabrics achieve near-zero packet loss through three cooperating mechanisms:

```text
Sender NIC (GPU 1) ──────► Leaf Switch (Queue builds) ──────► Receiver NIC (GPU 2)
                                │                                    │
   1. ECN threshold crossed ────┼──► Marks CE bits in IP header ────►│
                                │                                    │
   2. Receiver sends CNP ◄──────┼────────────────────────────────────┤ (Rate-limits sender)
                                │
   3. Emergency headroom hit ──►│──► Sends PFC PAUSE (802.1Qbb) ──► Upstream pauses priority class
```

### 1. ECN and DCQCN (Proactive congestion control)

- **Explicit Congestion Notification (ECN):** When a switch egress queue crosses a configured `Kmin` threshold, the ASIC marks the IP header's ECN field (`CE = 11`) instead of dropping the packet.
- **DCQCN (Data Center Quantized Congestion Notification):** The receiving RDMA NIC sees the CE mark and immediately sends a **Congestion Notification Packet (CNP)** back to the sender NIC, which smoothly throttles its transmission rate and later ramps back up.

### 2. PFC (Priority Flow Control — IEEE 802.1Qbb)

**PFC** is the **safety net**, not the primary congestion controller. It pauses a single CoS priority queue (typically priority 3 or 4 for RoCEv2) hop-by-hop when switch ingress buffer headroom is nearly full.

> [!CAUTION]
> **PFC storm / deadlock hazard:** Misconfigured PFC thresholds or a faulty NIC sending continuous `PAUSE` frames can propagate backpressure across the entire fabric (**victim flow starvation** and **PFC deadlock**). Always pair PFC with **ECN/DCQCN** (so ECN fires *before* PFC ever triggers) and enable **PFC Watchdog** on every switch port to drop/quarantine stuck queues.

### 3. Rail-optimized Clos topologies and load balancing

In a GPU server with 8 GPUs and 8 dedicated 400G/800G NICs (one NIC per GPU):

- **Rail-optimized design:** GPU 0 on every server connects to **Rail 0 Leaf**, GPU 1 on every server connects to **Rail 1 Leaf**, and so on. Because collective `AllReduce` rings primarily exchange data between matching GPU ranks across servers, most traffic stays **one hop away inside the same rail leaf** without ever crossing the spines.
- **Spray / adaptive load balancing:** Traditional 5-tuple ECMP hashing performs poorly on AI fabrics because a few huge RDMA elephant flows hash onto the same spine link (**hash polarization**). Modern AI Ethernet fabrics (and the Ultra Ethernet Consortium spec) use **packet spraying**, **adaptive routing (DLB/ARS)**, and NIC out-of-order reassembly to spread packets evenly across all spine paths.

## Summary

- Match optics to reach: **DAC** in-rack, **SMF (DR4/FR4/LR4)** for future-proof leaf-spine trunks; always verify **matching RS-FEC modes** and **DOM/DDM TX/RX power budgets**.
- Choose switch silicon to match the workload: **shallow shared-buffer cut-through** for uniform Clos fabrics; **deep-buffer VOQ** for speed-mismatch fan-in and WAN edges.
- Lossless **RoCEv2 AI/ML Ethernet fabrics** rely on **ECN/DCQCN** as the primary throttle, **PFC + PFC Watchdog** as the emergency safety net, and **rail-optimized Clos + adaptive load balancing** to eliminate elephant-flow collisions.
