# systemd-cgtop

## Overview

`systemd-cgtop` displays a live, `top`-like view of **systemd control groups (cgroups)** sorted by **CPU, memory, or disk I/O**. Unlike `top` or `htop`, which show individual processes, `systemd-cgtop` aggregates resource usage by **service, slice, or container** (`system.slice/nginx.service`, `user.slice`, `machine.slice`), making it effortless to see which service as a whole is saturating the host.

## Syntax

```bash
systemd-cgtop [options] [cgroup]
```

## Common Options

| Option | Description |
|--------|-------------|
| `-c` | Sort by CPU usage (default) |
| `-m` | Sort by memory usage |
| `-i` | Sort by disk I/O |
| `-t` | Sort by number of tasks/threads |
| `-p` | Sort by control group path |
| `-n N` | Run for `N` iterations and exit (great for `-n 1` snapshots) |
| `-b` | Batch mode (non-interactive, for scripts/logs) |
| `-d SEC` | Refresh interval in seconds |
| `--depth=N` | Limit cgroup tree depth (default is `3`) |

## Key Use Cases

1. Identify which **service** (including all its worker processes) is eating RAM or CPU
2. Monitor container/VM slices (`machine.slice`, `user.slice`) at a glance
3. Take a one-shot cgroup resource snapshot in incident triage (`systemd-cgtop -b -n 1`)

## Examples with Explanations

### Watch cgroups sorted by memory usage

```bash
systemd-cgtop -m
```

Shows total memory per unit (e.g., `system.slice/postgresql.service`), including all child workers.

### One-shot non-interactive snapshot sorted by I/O

```bash
systemd-cgtop -b -n 1 -i
```

Useful when capturing diagnostic bundles or piping into `head`.

### Focus on a specific slice

```bash
systemd-cgtop /system.slice
```

## Notes & Pitfalls

- If `CPU` or `Input/Output` columns show `-`, the corresponding cgroup accounting controller (`CPUAccounting=yes`, `IOAccounting=yes`) may not be enabled globally or on that unit (modern systemd enables `CPUAccounting` and `MemoryAccounting` by default on cgroup v2).
- Memory reported by `systemd-cgtop` reflects the cgroup's current memory accounting (including page cache charged to that cgroup), which can differ from `ps` RSS.

## Related Commands

- `systemctl` — inspect and set unit resource limits (`MemoryMax=`, `CPUQuota=`)
- `htop` / `top` — per-process resource monitoring
- `systemd-analyze` — boot time and unit verification

## Additional Resources

- `man systemd-cgtop`
- `man systemd.resource-control`
