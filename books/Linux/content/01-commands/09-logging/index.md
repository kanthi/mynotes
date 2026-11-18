---
title: Intro
---

# Intro

Write, inspect, rotate, and audit Linux logs: kernel ring buffer (`dmesg`), syslog emission (`logger`), rotation (`logrotate`), interactive analysis (`lnav`), and kernel security auditing (`auditctl`/`ausearch`). On systemd systems, pair these with `journalctl` from Services and runtime.

## Commands in this part

| Command | Role |
|---------|------|
| `dmesg` | dmesg prints the kernel ring buffer: hardware detection, driver messages, OOM kills, I/O errors, and other kernel events. |
| `logger` | logger writes messages to the system log from the shell or scripts. |
| `logrotate` | logrotate rotates, compresses, and retires log files so they do not fill disks. |
| `lnav` | lnav is an interactive terminal log viewer that merges multiple logs chronologically, reads `.gz` archives, and supports SQL queries. |
| `auditctl` / `ausearch` | auditctl configures kernel audit rules (`auditd`), and ausearch queries `/var/log/audit/audit.log` for file changes and syscalls. |

## Pipeline & Triage Recipes

```bash
# 1. Check for OOM kills or disk errors in the kernel ring buffer
dmesg -T --level=err,warn | tail -n 30

# 2. Correlate system logs, web errors, and rotated .gz files in one timeline
sudo lnav -r /var/log/syslog /var/log/nginx/error.log

# 3. Emit a structured marker from a deploy script to syslog/journald
logger -t deploy -p user.info "Released build v2.4.1"
```

## Suggested starting points

1. Kernel messages: `dmesg` (and `journalctl -k`).
2. Emit a message from scripts: `logger`.
3. Manage disk growth: `logrotate`.
4. Interactive triage & SQL over logs: `lnav`.
5. Track who changed sensitive files: `auditctl` / `ausearch`.

## Related parts

- Services and runtime — `journalctl`
- Text and pipes — mine log files with `grep`/`jq`/`awk`
- Archives and compression — `zgrep` and `zcat` on rotated `.gz` logs

Continue with the individual command pages in this part.
