# nethogs

## Overview

`nethogs` is a live `top`-like network monitor that groups **network bandwidth by process and PID** instead of by interface or port. When `sar` or `btop` shows your NIC is saturated at 1 Gbps, `nethogs` immediately tells you *which binary* (`rsync`, `curl`, `apt`, `python3`) is sending or receiving the traffic.

```bash
sudo apt install nethogs   # Debian/Ubuntu
sudo dnf install nethogs   # Fedora/RHEL
```

## Syntax

```bash
sudo nethogs [options] [device...]
```

## Common Options & Interactive Keys

| Option / Key | Description |
|--------------|-------------|
| `[device...]` | Monitor specific interface(s) (e.g., `eth0`, `ens5`) |
| `-t` | Trace mode (print plain text updates line by line—great for logging) |
| `-c N` | Exit after `N` refreshes (pair with `-t`) |
| `-d SEC` | Refresh delay in seconds (default `1`) |
| `-v MODE` | View mode: `0` = KB/s, `1` = total KB, `2` = total B, `3` = total MB, `4` = MB/s |
| `-a` | Monitor all interfaces including loopback (`lo`) |
| `s` / `r` | *(Interactive)* Sort by Sent (`s`) or Received (`r`) traffic |
| `m` | *(Interactive)* Cycle units (KB/s, KB, B, MB) |

## Key Use Cases

1. Find out which process is saturating upload or download bandwidth right now
2. Measure total bytes transferred by a process during a test (`-v 1` or `m`)
3. Log short bandwidth bursts non-interactively (`sudo nethogs -t -c 10 eth0`)

## Safety

- Requires `sudo` (or `CAP_NET_RAW` + `CAP_NET_ADMIN`) to capture raw socket packets.
- Because `nethogs` inspects packet headers via `libpcap`, running it on extremely high-throughput 10GbE+ links adds some CPU overhead.

## Examples with Explanations

### Monitor all default interfaces live

```bash
sudo nethogs
```

### Monitor a specific NIC in MB/s (`-v 4`)

```bash
sudo nethogs -v 4 eth0
```

### Capture a 10-second trace log in scripts

```bash
sudo nethogs -t -d 1 -c 10 eth0 > /tmp/nethogs-trace.log
```

### Include loopback traffic (`-a`) for local DB/API profiling

```bash
sudo nethogs -a lo
```

## Notes & Pitfalls

- Kernel-space traffic (such as NFS kernel client or WireGuard/VPN encapsulation) or short-lived connections that close before the refresh tick may show under `unknown TCP` instead of a PID.
- `nethogs` shows **per-process** bandwidth; for **per-remote-IP/port** bandwidth breakdowns, use `iftop` or `tcpdump`.

## Related Commands

- `ss` — inspect active sockets and owning PIDs (`ss -tunap`)
- `lsof` (`lsof -i`) — open network sockets by process
- `tcpdump` — packet-level capture and filtering
- `iperf3` — active network throughput benchmarking

## Additional Resources

- `man nethogs`
