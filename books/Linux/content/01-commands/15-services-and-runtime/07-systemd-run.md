# systemd-run

## Overview

`systemd-run` creates **transient** systemd units in memory on the fly—one-shot commands, background services, interactive scopes, or temporary timers—without writing permanent `.service` or `.timer` files under `/etc/systemd/system/`. It is ideal for running resource-capped jobs (`CPUQuota=`, `MemoryMax=`), ad-hoc scheduled tasks, and testing cgroup/sandbox directives before committing them to unit files.

## Syntax

```bash
systemd-run [options] command [args...]
systemd-run --user [options] command [args...]
```

## Common Options

| Option | Description |
|--------|-------------|
| `--unit=NAME` | Assign a readable unit name instead of an auto-generated `run-r...` name |
| `-p PROP=VAL` / `--property=` | Set unit/cgroup properties (`MemoryMax=512M`, `CPUQuota=50%`, `IOWeight=50`) |
| `--on-calendar=SPEC` | Create a transient timer on a wall-clock schedule |
| `--on-active=SPAN` | Run once after a relative delay (e.g., `15m`, `2h`) |
| `--wait` | Block until the transient service finishes and print its exit status |
| `--collect` | Automatically garbage-collect the unit after exit (even on failure) |
| `-t` / `--pty` | Allocate a pseudo-terminal for interactive commands |
| `-P` / `--pipe` | Connect stdin/stdout/stderr directly to the calling shell |
| `--uid=USER` / `--gid=GROUP` | Run the command as a specific user or group |
| `--working-directory=DIR` | Set the working directory (`--same-dir` uses current `$PWD`) |

## Key Use Cases

1. Cap CPU, RAM, or disk I/O of a heavy one-off job (`-p CPUQuota=50% -p MemoryMax=1G`)
2. Schedule a quick one-shot or recurring transient timer without editing `/etc/systemd/system/`
3. Test `systemd` sandboxing (`ProtectSystem=strict`, `PrivateTmp=yes`) interactively before putting it in a unit file
4. Run a background command cleanly detached in its own cgroup instead of `nohup`

## Safety

- Transient units and timers live in memory and **disappear on reboot**—convert them to real unit files if they must persist.
- Setting tight `MemoryMax=` values triggers the cgroup OOM killer immediately if exceeded; test with `--user` or `MemoryHigh=` first.
- System-level transient units require `sudo`; `--user` works unprivileged for your own session.

## Examples with Explanations

### Run a background job with a custom name and wait for it

```bash
sudo systemd-run --wait --unit=scratch-backup /usr/local/bin/backup.sh
systemctl status scratch-backup
journalctl -u scratch-backup -n 30 --no-pager
```

### Cap CPU and memory for a heavy batch job

```bash
sudo systemd-run -p CPUQuota=50% -p MemoryMax=512M -p IOWeight=20 \
  --unit=video-encode --wait --collect /usr/local/bin/encode.sh
```

Prevents an encoding or compression job from starving web services on the same host.

### Schedule a one-shot delayed task (modern `at` replacement)

```bash
sudo systemd-run --on-active=15m --unit=restart-worker systemctl restart worker.service
systemctl list-timers | grep restart-worker
```

### Create a transient recurring calendar timer

```bash
sudo systemd-run --on-calendar='*-*-* 03:00:00' --unit=nightly-clean \
  /usr/local/bin/cleanup-tmp.sh
```

### Open an interactive shell inside a resource-capped cgroup

```bash
sudo systemd-run --pty --same-dir --wait --collect -p MemoryMax=256M bash
```

### Run a user-level transient notification

```bash
systemd-run --user --on-active=10m notify-send 'Build check' 'Verify staging deploy'
```

## Notes & Pitfalls

- If a transient unit fails without `--collect`, it stays in `failed` state until `systemctl reset-failed <unit>` (helpful for reading logs, noisy in `systemctl --failed` if forgotten).
- Use `systemd-run --scope` instead of a service when you want the command to run synchronously in the caller's environment while still being tracked in its own cgroup.

## Related Commands

- `systemctl` — inspect, stop, or reset transient units
- `systemd-cgtop` — watch live CPU/RAM/I/O of the transient unit's cgroup
- `systemd-analyze` — validate `--on-calendar=` expressions
- `at` / `cron` — classic non-systemd scheduling

## Additional Resources

- `man systemd-run`
- `man systemd.resource-control`
