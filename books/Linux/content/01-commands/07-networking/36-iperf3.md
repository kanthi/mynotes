# iperf3

## Overview

`iperf3` measures **maximum achievable network bandwidth, jitter, and packet loss** between two hosts over TCP, UDP, or SCTP. Unlike `ping` (which only measures latency) or `curl` (which includes disk/HTTP overhead), `iperf3` tests raw memory-to-memory network throughput—making it the standard benchmark for verifying 1GbE/10GbE/25GbE links, VLANs, VPNs, and cloud VPC peering.

```bash
sudo apt install iperf3   # Debian/Ubuntu
sudo dnf install iperf3   # Fedora/RHEL
```

## Syntax

```bash
# On the receiving server:
iperf3 -s [options]

# On the sending client:
iperf3 -c <server_ip> [options]
```

## Common Options

| Option | Description |
|--------|-------------|
| `-s` | Run in server mode (listens on TCP/UDP port `5201` by default) |
| `-c HOST` | Run in client mode connecting to `HOST` |
| `-p PORT` | Use a custom port instead of `5201` |
| `-t SEC` | Test duration in seconds (default `10`) |
| `-P N` | Run `N` parallel TCP streams (essential to saturate 10GbE+ links) |
| `-R` | Reverse mode: **server sends, client receives** (tests download speed from the server) |
| `-u` | Use UDP instead of TCP (measures jitter and packet loss) |
| `-b RATE` | Target bandwidth (e.g., `-b 1G` for UDP, or `-b 500M` to cap TCP) |
| `-J` | Output results in JSON format |
| `-1` | Server exits after handling one client test (great for ad-hoc checks) |

## Key Use Cases

1. Verify whether a 10GbE NIC or cloud instance actually delivers advertised bandwidth (`-P 4`)
2. Test both upload and download paths from one side using `-R` (reverse mode)
3. Measure UDP jitter and packet loss (`-u -b 100M`) for VoIP/streaming or WireGuard tunnels
4. Compare MTU/offload tuning changes objectively before and after (`-J` JSON output)

## Safety

- An `iperf3` test will **saturate the network link** by default for 10 seconds—avoid running unconstrained tests on live production links during peak traffic (or cap with `-b 500M`).
- Ensure TCP/UDP port `5201` is open between client and server (or tunnel/restrict it via firewall rules) and stop `iperf3 -s` when done so it doesn't stay exposed.

## Examples with Explanations

### Basic 10-second TCP throughput test

```bash
# Server:
iperf3 -s -1

# Client:
iperf3 -c 10.0.1.20
```

`-1` automatically shuts down the server after the test finishes.

### Saturate a high-speed 10GbE / 25GbE link with parallel streams

```bash
iperf3 -c 10.0.1.20 -P 8 -t 15
```

A single TCP stream often bottlenecks on one CPU core or TCP window size; `-P 8` uses 8 parallel streams to measure true link capacity.

### Test download speed (server -> client) with `-R`

```bash
iperf3 -c 10.0.1.20 -R -P 4
```

Measures inbound throughput to your client without needing to swap client/server roles.

### Measure UDP jitter and packet loss

```bash
iperf3 -c 10.0.1.20 -u -b 100M -t 10
```

Look at the `Jitter` (ms) and `Lost/Total Datagrams` (`%`) columns at the end.

### Bidirectional test and JSON output

```bash
iperf3 -c 10.0.1.20 --bidir -J > /tmp/iperf-result.json
```

`--bidir` tests send and receive simultaneously (supported in `iperf3` 3.7+).

### Run `iperf3` over an SSH tunnel (when port 5201 is firewalled)

```bash
ssh -f -N -L 5201:127.0.0.1:5201 user@10.0.1.20
iperf3 -c 127.0.0.1
```

*(Note: SSH encryption caps throughput to single-core cipher speed, so use direct TCP for raw NIC benchmarks.)*

## Understanding Output

In TCP output, watch two key columns:
- **`Bitrate` (`sender` vs `receiver`)**: Sustained throughput in `Gbits/sec` or `Mbits/sec`.
- **`Retr` (Retransmits)**: Non-zero retransmissions indicate packet loss, bufferbloat, duplex mismatch, or MTU fragmentation.

## Notes & Pitfalls

- `iperf3` is **not** compatible with older `iperf` (v2) clients/servers.
- Single-threaded `iperf3` instances share one server process per port; for 40GbE/100GbE testing, run multiple `iperf3 -s -p 5201` / `5202` processes pinned to NUMA cores (`taskset`).

## Related Commands

- `ethtool` — check physical NIC link speed (`1000Mb/s` vs `10000Mb/s`) and CRC errors
- `mtr` / `ping` — diagnose latency and hop-by-hop packet loss
- `nethogs` — watch live per-process bandwidth
- `ss` — inspect TCP socket buffers and congestion control (`ss -ti`)

## Additional Resources

- `man iperf3`
