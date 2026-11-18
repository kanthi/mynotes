# pidstat

## Overview

`pidstat` (from the `sysstat` package) reports **per-process CPU, memory, page faults, context switches, and disk I/O** at regular intervals. Unlike `top` (which clears the screen) or `ps` (which only gives a cumulative average since boot), `pidstat` prints clean, timestamped, appendable rows—ideal for correlating spikes across processes in scripts and incident logs.

```bash
sudo apt install sysstat   # Debian/Ubuntu
sudo dnf install sysstat   # Fedora/RHEL
```

## Syntax

```bash
pidstat [options] [interval [count]]
```

## Common Options

| Option | Description |
|--------|-------------|
| `-u` | Report CPU utilization (`%usr`, `%system`, `%wait`, `%CPU`; default) |
| `-r` | Report memory utilization (`minflt/s`, `majflt/s`, `VSZ`, `RSS`, `%MEM`) |
| `-d` | Report disk I/O statistics (`kB_rd/s`, `kB_wr/s`, `iodelay`) |
| `-w` | Report voluntary (`cswch/s`) and involuntary (`nvcswch/s`) context switches |
| `-t` | Include individual threads (`TGID` and `TID`) |
| `-p PID\|ALL` | Filter by PID (or `ALL`) |
| `-C COMM` | Filter by command name regex (e.g., `-C nginx`) |
| `-h` | Horizontal single-line format (easy to parse with `awk`) |
| `-l` | Print full command line arguments |

## Key Use Cases

1. Track **which thread** inside a multi-threaded Java/Go/C++ process is pegging 100% CPU (`pidstat -t -p PID 1`)
2. Measure per-process disk I/O (`-d`) and I/O wait delays (`iodelay`) over time
3. Spot memory thrashing via major page faults (`majflt/s` with `-r`)
4. Detect lock contention or CPU starvation via context switches (`-w`)

## Examples with Explanations

### Sample all active processes every 2 seconds (CPU)

```bash
pidstat 2 5
```

Prints 5 intervals of 2 seconds each, showing only processes with non-zero CPU activity.

### Inspect threads of a busy process (`-t`)

```bash
pidstat -t -p "$(pgrep -x java)" 1 5
```

Reveals the exact `TID` consuming CPU (`%usr` vs `%system`), which you can match against a `jstack` or `perf` thread dump.

### Monitor per-process disk I/O and `iodelay`

```bash
sudo pidstat -d 2
```

Shows `kB_rd/s`, `kB_wr/s`, `kB_ccwr/s` (cancelled writes, e.g., temp files deleted before flush), and `iodelay` (clock ticks waiting for block I/O/swap).

### Watch memory growth and major page faults (`-r`)

```bash
pidstat -r -C python3 2
```

High `majflt/s` means the process is reading pages from disk or swap instead of RAM.

### Combine CPU, memory, and I/O in one horizontal line (`-urd -h`)

```bash
sudo pidstat -urd -h -p "$(pgrep -d, -x postgres)" 5
```

## Understanding Output

In `pidstat -w`:

| Column | Meaning |
|--------|---------|
| `cswch/s` | **Voluntary** context switches (process yielded waiting for I/O, mutex, or sleep) |
| `nvcswch/s` | **Involuntary** context switches (scheduler preempted the process because its CPU slice expired—high values signal CPU contention) |

## Notes & Pitfalls

- Running `pidstat` **without** an interval (`pidstat`) prints averages since system boot, which hides recent spikes—always pass an interval like `pidstat 1 5`.
- Reading `/proc/PID/io` for other users' processes (`-d`) requires `sudo`.

## Related Commands

- `mpstat` / `iostat` / `vmstat` / `sar` — sibling `sysstat` system-wide monitors
- `iotop` — interactive real-time disk I/O viewer
- `top` / `htop` — interactive process dashboards

## Additional Resources

- `man pidstat`
