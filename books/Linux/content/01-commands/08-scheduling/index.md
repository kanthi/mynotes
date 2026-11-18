---
title: Intro
---

# Intro

Run work later or periodically. Classic `cron`/`crontab`/`anacron`/`at` remain everywhere; on systemd hosts, `systemd` timers (`systemctl list-timers`, `.timer` units) provide centralized logging, dependency ordering, and missed-run catch-up.

## Commands in this part

| Command | Role |
|---------|------|
| `cron` | cron is the classic daemon that runs scheduled jobs from per-user and system crontab files. |
| `crontab` | crontab installs, lists, and removes per-user crontab files that the cron daemon executes. |
| `anacron` | anacron runs periodic commands on machines that are not on 24/7 — laptops, desktops, and intermittent VMs. |
| `at` | at schedules a one-shot command to run once at a future time. |
| `systemd` timers | `.timer` units trigger `.service` jobs on calendar or monotonic schedules with `journalctl` logging and `Persistent=true` catch-up. |

## Which tool should I use?

| Need | Best Choice | Why |
|------|-------------|-----|
| Simple per-user recurring job on any Unix host | `crontab` | Universal, minimal setup |
| Modern Linux service job with logs, jitter, or dependencies | `systemd` timers | Native `journalctl` logs, `Persistent=true`, `RandomizedDelaySec=` |
| Daily/weekly job on a laptop or intermittent VM (non-systemd) | `anacron` | Catches up after power-off |
| One-shot command later today | `at` or `systemd-run` | `at` is simple; `systemd-run` integrates with systemd logs |

## Suggested starting points

1. User jobs: `crontab -e` / `crontab -l`.
2. Daemon and system tables: `cron`, `/etc/cron.d`.
3. Machines that sleep: `anacron` or `systemd` timers with `Persistent=true`.
4. One-shot future jobs: `at` or `systemd-run --on-active=`.
5. Modern default on systemd hosts: `systemd` timers (`systemctl list-timers`).

## Related parts

- Services and runtime — `systemd-run`, `systemctl`, `journalctl`
- Logging — capture job output with `logger` / `journalctl`
- Shell commands — reliable one-liners for jobs

Continue with the individual command pages in this part.
