# renice

## Overview

`renice` changes the **CPU scheduling priority (niceness)** of **already-running** processes, process groups, or all processes owned by a user. While `nice` sets priority when launching a new command, `renice` lets you throttle a runaway build, backup, or compression job on a live system without restarting it.

## Syntax

```bash
renice [-n] priority [-g | -p | -u] identifier...
```

## Common Options

| Option | Description |
|--------|-------------|
| `-n PRIORITY` | Target nice value from `-20` (highest CPU priority) to `19` (lowest CPU priority); default is `0` |
| `-p PID...` | Interpret following arguments as Process IDs (default) |
| `-u USER...` | Adjust all processes owned by `USER` |
| `-g PGID...` | Adjust all processes in process group `PGID` |

## Key Use Cases

1. Lower the priority of a CPU-heavy `tar`, `ffmpeg`, or compile job (`renice -n 19 -p PID`) so SSH and web services stay responsive
2. Boost a critical interactive or diagnostic process (`sudo renice -n -5 -p PID`) during high load
3. Deprioritize all processes belonging to a specific batch user (`sudo renice -n 15 -u batchuser`)

## Safety

- Unprivileged users can only **increase** niceness (make their own processes *lower* priority, up to `19`); lowering niceness below the current value (or toward `-20`) or modifying another user's processes requires `sudo`.
- Setting `-20` on a tight CPU loop can starve critical system daemons and SSH—use `-5` or `-10` sparingly.

## Examples with Explanations

### Lower priority of a running backup or compression job

```bash
pgrep -a zstd
renice -n 19 -p 4821
```

Makes PID `4821` yield CPU whenever higher-priority tasks need it.

### Boost priority of an urgent process (requires `sudo`)

```bash
sudo renice -n -5 -p 1234
```

### Deprioritize all processes of a specific user

```bash
sudo renice -n 15 -u renderbot
```

### Throttle an entire process group

```bash
sudo renice -n 10 -g 4800
```

### Combine with `ionice` to tame a heavy job on both CPU and disk

```bash
sudo renice -n 19 -p 4821
sudo ionice -c 3 -p 4821
```

Sets both lowest CPU priority (`19`) and `idle` disk I/O class (`-c 3`).

## Understanding Output

`renice` prints the old and new priority for each target:

```text
4821 (process ID) old priority 0, new priority 19
```

## Notes & Pitfalls

- `renice -n 10` sets the **absolute** nice value to `10` on modern `util-linux` (not `+10` relative to the current value—check `man renice` if using an older/BSD implementation).
- `renice` only adjusts **CPU** scheduling; if a process is saturating disk I/O, pair it with `ionice`.
- For persistent service limits across restarts, configure `Nice=` or `CPUWeight=` in the systemd unit file instead.

## Related Commands

- `nice` — launch a command with an initial nice value
- `ionice` — adjust disk I/O scheduling class and priority
- `chrt` — set real-time scheduling policies (`SCHED_FIFO`, `SCHED_RR`)
- `top` / `htop` — inspect `NI` column and renice interactively (`r` / `F7`/`F8`)

## Additional Resources

- `man renice`
- `man 7 sched`
