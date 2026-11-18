# iotop

## Overview

`iotop` is a live `top`-like monitor for **disk I/O per process and thread**. While `iostat` tells you *which disk* is 100% busy, `iotop` tells you *which exact process* is hammering the disk with reads, writes, or `fsync` stalls. Requires root (or `CAP_NET_ADMIN`).

```bash
sudo apt install iotop-c   # modern C rewrite on Debian/Ubuntu (or iotop)
sudo dnf install iotop     # Fedora/RHEL
```

## Syntax

```bash
sudo iotop [options]
```

## Common Options & Interactive Keys

| Option / Key | Description |
|--------------|-------------|
| `-o`, `--only` | Show only processes/threads **actually doing I/O** right now |
| `-a`, `--accumulated` | Show **accumulated** bytes read/written since `iotop` started (instead of current rate) |
| `-P`, `--processes` | Show processes only (hide individual kernel/app threads) |
| `-p PID` | Monitor only specific PID(s) |
| `-u USER` | Monitor only processes owned by `USER` |
| `-b`, `--batch` | Non-interactive batch mode (for logging to files) |
| `-n N` / `-d SEC` | Exit after `N` iterations; refresh every `SEC` seconds |
| `-t` / `-q` | Add timestamps (`-t`) and suppress repeating headers (`-q`) in batch mode |
| `o` / `a` / `p` | *(Interactive)* Toggle `--only`, `--accumulated`, and `--processes` |
| `Left` / `Right` | *(Interactive)* Change sort column (`DISK READ`, `DISK WRITE`, `SWAPIN`, `IO>`) |

## Key Use Cases

1. Immediately identify which process is saturating disk bandwidth (`iotop -oP`)
2. Catch noisy background jobs (`updatedb`, `rsync`, `journald`, `apt`, `postgres` checkpoints)
3. Log short I/O spikes in batch mode (`iotop -botq -n 10`) for post-mortem analysis
4. Spot processes stalling on **swap-in** (`SWAPIN` column)

## Safety

- `iotop` reads `/proc/PID/io` and kernel taskstats via netlink—overhead is low, though in batch mode on thousands of threads `-P` reduces noise and work.

## Examples with Explanations

### Live view of only active I/O processes

```bash
sudo iotop -oP
```

Hides idle processes and collapses threads so the culprit jumps out immediately.

### Track total accumulated I/O over a few minutes

```bash
sudo iotop -oaP
```

Great for catching short, bursty writes (like periodic `fsync` spikes) that flash by too fast in rate mode.

### Watch I/O for a specific user or PID

```bash
sudo iotop -o -u postgres
sudo iotop -o -p "$(pgrep -d, -x mysqld)"
```

### Capture a timestamped batch log for intermittent I/O stalls

```bash
sudo iotop -botq -d 2 -n 30 > /tmp/iotop.log
```

Records 30 samples (every 2 seconds) of active I/O processes with timestamps.

### Pair with `ionice` to throttle an offender

```bash
# Found PID 4821 saturating writes in iotop:
sudo ionice -c 3 -p 4821
```

## Understanding Output

| Column | Meaning |
|--------|---------|
| `TID` / `PID` | Thread or Process ID |
| `PRIO` | I/O scheduling class/priority (`be/4` = best-effort 4, `idle` = class 3) |
| `DISK READ` / `DISK WRITE` | Current (or accumulated `-a`) disk throughput |
| `SWAPIN` | Percentage of time spent waiting on swap-in |
| `IO>` | Percentage of time spent waiting on I/O (default sort column) |

## Notes & Pitfalls

- If `iotop` complains `CONFIG_TASK_DELAY_ACCT` is disabled or `IO>` shows `0.00%`, check `sysctl kernel.task_delayacct=1` (newer kernels disable delay accounting by default to save minor CPU overhead unless enabled).
- Writes to page cache (`write()`) may initially be credited to the app, while actual disk flushes later show under kernel `kworker` / `jbd2` / `btrfs-transacti` threads.

## Related Commands

- `iostat` — per-device queue depth (`aqu-sz`), `await`, and `%util`
- `pidstat` (`pidstat -d`) — historical/interval per-PID disk I/O from `sysstat`
- `ionice` — throttle a process's disk I/O priority
- `lsof` — see which files a busy process has open

## Additional Resources

- `man iotop`
