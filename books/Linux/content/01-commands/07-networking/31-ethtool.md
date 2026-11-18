# ethtool

## Overview

`ethtool` queries and configures **Ethernet network interface (NIC)** hardware and driver settings: physical link detection, negotiated speed and duplex, driver/firmware versions, hardware offloads (`TSO`, `GRO`, `LRO`), ring buffers, and per-NIC error counters. Use it whenever `ip link` shows an interface exists, but link speed, packet drops, or physical cabling are suspect.

```bash
sudo apt install ethtool   # Debian/Ubuntu
sudo dnf install ethtool   # Fedora/RHEL
```

## Syntax

```bash
ethtool [options] devname
sudo ethtool -s devname [speed N] [duplex half|full] [autoneg on|off]
```

## Common Options

| Option | Description |
|--------|-------------|
| *(no flag)* | Display link status, supported/advertised speeds, duplex, and auto-negotiation |
| `-i` | Show kernel driver name, driver version, firmware version, and PCI bus address |
| `-S` | Print detailed NIC/driver hardware statistics (CRC errors, RX/TX drops, misses) |
| `-k` / `-K` | Query (`-k`) or toggle (`-K`) hardware offload features (`gro`, `tso`, `gso`, `rx`) |
| `-g` / `-G` | Query (`-g`) or resize (`-G`) RX/TX ring buffers |
| `-s` | Set speed, duplex, auto-negotiation, or Wake-on-LAN (`wol`) |
| `-p [SEC]` | Blink the physical port's LED for `SEC` seconds to identify which cable is `eth0` |

## Key Use Cases

1. Confirm physical link state (`Link detected: yes`) and negotiated speed (`1000Mb/s` vs `10000Mb/s` vs `100Mb/s` fallback)
2. Blink a server NIC LED (`ethtool -p`) when tracing cables in a rack
3. Check NIC hardware RX/TX drop and CRC error counters (`ethtool -S`)
4. Inspect driver and firmware versions (`ethtool -i`) before reporting kernel/NIC bugs

## Safety

- Forcing speed/duplex with `autoneg off` can cause a duplex mismatch and severe packet loss if the switch port is still auto-negotiating—keep `autoneg on` unless specifically required.
- Toggling offloads (`-K`) or ring buffers (`-G`) resets the NIC queue briefly and can drop active packets—test during maintenance windows.
- Changes made with `ethtool` are **not persistent** across reboots unless configured in `netplan`, NetworkManager (`nmcli`), `systemd-networkd` (`.link` files), or a udev rule.

## Examples with Explanations

### Check link speed, duplex, and cable detection

```bash
ethtool eth0
```

Look for `Speed:`, `Duplex: Full`, `Auto-negotiation: on`, and `Link detected: yes`.

### Identify which physical RJ45/SFP port corresponds to `enp3s0`

```bash
sudo ethtool -p enp3s0 10
```

Blinks the port LED for 10 seconds so you can match it in the server rack.

### Inspect NIC hardware error and drop counters

```bash
ethtool -S eth0 | grep -iE 'err|drop|crc|miss|fifo'
```

Non-zero CRC or RX missed counters point to bad cables, dirty SFP optics, or undersized RX ring buffers.

### Check driver and firmware info

```bash
ethtool -i eth0
```

### Inspect and resize RX/TX ring buffers

```bash
ethtool -g eth0
sudo ethtool -G eth0 rx 4096
```

Increasing RX ring size helps prevent burst packet drops on busy 10GbE/25GbE hosts.

### Toggle an offload feature for debugging

```bash
ethtool -k eth0 | grep -E 'generic-receive-offload|tcp-segmentation-offload'
sudo ethtool -K eth0 gro off
```

## Understanding Output

In `ethtool <dev>`:

| Field | What to Check |
|-------|---------------|
| `Supported link modes` | Maximum speeds the NIC hardware supports |
| `Advertised link modes` | Speeds offered during auto-negotiation |
| `Speed` / `Duplex` | Actual negotiated link (e.g., `100Mb/s` on a `1000Mb/s` port signals a bad cable/pin) |
| `Link detected` | Physical carrier (`yes` = cable/optic connected and active) |

## Notes & Pitfalls

- Virtual NICs (`virtio_net`, `veth` in containers) report limited or synthetic `ethtool` output (e.g., `Speed: Unknown!`).
- Check `ethtool --show-eee` if Energy-Efficient Ethernet causes micro-latency spikes on low-latency links.

## Related Commands

- `ip` (`ip -s link`) — interface state and kernel-level RX/TX counters
- `nmcli` — persist NIC settings in NetworkManager
- `lspci` — identify the underlying NIC chipset
- `dmesg` — check kernel link-up/down and SFP transceiver messages

## Additional Resources

- `man ethtool`
