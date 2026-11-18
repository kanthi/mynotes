# systemd timers

## Overview

`systemd` timers are `.timer` unit files that trigger a matching `.service` unit on a calendar schedule or monotonic interval. On modern Linux distributions, they replace `cron` and `anacron` for system jobs because they integrate with `journalctl` logging, respect service dependencies, support randomized jitter, and can catch up on missed runs after downtime (`Persistent=true`).

## Syntax

```bash
systemctl list-timers [options]
systemd-analyze calendar "SPEC"
systemd-run --on-calendar="SPEC" command
```

## Common Commands & Flags

| Command / Option | Description |
|------------------|-------------|
| `systemctl list-timers` | List all active timers with next/last run times |
| `systemctl list-timers --all` | Include inactive or stopped timers |
| `systemctl enable --now foo.timer` | Enable a timer at boot and start it immediately |
| `systemctl start foo.service` | Manually trigger the underlying service right now for testing |
| `systemctl status foo.timer` | Check timer state, trigger schedule, and next activation |
| `journalctl -u foo.service` | View stdout/stderr logs from past timer runs |
| `systemd-analyze calendar "..."` | Validate and preview `OnCalendar=` expressions |

## Key Use Cases

1. Run recurring maintenance or backups with built-in `journalctl` logging
2. Catch up on missed jobs after a server reboot or laptop sleep (`Persistent=true`)
3. Stagger fleet-wide jobs with `RandomizedDelaySec=` to avoid thundering herds
4. Test calendar expressions before deploying to production

## Safety

- Always validate `OnCalendar=` strings with `systemd-analyze calendar`—a typo can prevent a timer from firing or run it far more often than intended.
- Enabling a `.timer` does **not** require enabling its matching `.service` (`[Install]` goes on the `.timer` unit).
- Use `OnBootSec=` or `OnUnitActiveSec=` for intervals, and `OnCalendar=` for wall-clock schedules.

## Examples with Explanations

### Inspect active timers and upcoming runs

```bash
systemctl list-timers --all
```

Shows `NEXT`, `LEFT`, `LAST`, `PASSED`, `UNIT`, and `ACTIVATES` (the `.service` triggered).

### Validate a calendar expression before saving

```bash
systemd-analyze calendar "Mon..Fri 02:30:00"
systemd-analyze calendar "*-*-* 00/4:00:00"
```

Prints the normalized form, next elapse time, and countdown—essential before putting a schedule into a unit file.

### Create a persistent daily backup timer

Pair `/etc/systemd/system/db-backup.service`:

```ini
[Unit]
Description=Daily database backup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup-db.sh
```

With `/etc/systemd/system/db-backup.timer`:

```ini
[Unit]
Description=Run daily database backup at 03:15

[Timer]
OnCalendar=*-*-* 03:15:00
RandomizedDelaySec=10m
Persistent=true

[Install]
WantedBy=timers.target
```

Reload and activate:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now db-backup.timer
```

`Persistent=true` ensures that if the machine was off at 03:15, the backup runs shortly after boot. `RandomizedDelaySec=10m` spreads load across multiple servers.

### Test the job immediately and check its logs

```bash
sudo systemctl start db-backup.service
journalctl -u db-backup.service -n 50 --no-pager
```

Lets you verify the script works without waiting for the timer to fire.

### Schedule a quick transient timer without writing unit files

```bash
sudo systemd-run --on-active=15m --unit=restart-cache systemctl restart redis
```

Creates an in-memory timer that runs once in 15 minutes (similar to `at`, but tracked by systemd).

### User-level timers (no root required)

```bash
systemctl --user list-timers
systemctl --user enable --now my-sync.timer
```

Reads units from `~/.config/systemd/user/`. Note that user timers only run while logged in unless lingering is enabled (`loginctl enable-linger "$USER"`).

## Understanding Output

`systemctl list-timers` columns:

| Column | Meaning |
|--------|---------|
| `NEXT` | Timestamp of the next scheduled trigger |
| `LEFT` | Time remaining until `NEXT` |
| `LAST` | Timestamp of the most recent trigger (`n/a` if never run) |
| `PASSED` | Time elapsed since `LAST` |
| `UNIT` | Name of the `.timer` unit |
| `ACTIVATES` | Target `.service` unit that runs when triggered |

## Notes & Pitfalls

- `AccuracySec=` defaults to `1min`, meaning systemd may coalesce timers within a 1-minute window to save power; set `AccuracySec=1s` if sub-minute precision matters.
- Unlike `cron`, systemd timers do **not** email output by default—everything goes to the journal (`journalctl -u unit.service`).
- If a `Type=oneshot` service is still running when the next timer tick arrives, systemd waits for it to finish rather than launching overlapping copies.

## Related Commands

- `cron` / `crontab` — classic fixed-schedule daemon
- `anacron` — classic catch-up scheduler for non-24/7 hosts
- `at` — one-shot delayed execution
- `systemd-run` — create transient services and timers on the fly
- `journalctl` — inspect timer and service logs

## Additional Resources

- `man systemd.timer`
- `man systemd.time`
- `man systemd-analyze`
