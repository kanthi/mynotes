# auditctl & ausearch

## Overview

`auditctl` and `ausearch` are the core user-space tools for the **Linux Audit framework (`auditd`)**. While `dmesg` and `logger` record kernel and application messages, `auditd` logs **security-relevant kernel events**—such as who modified `/etc/passwd`, which process executed a binary, or which syscall failed with `EACCES`/`EPERM`. `auditctl` manages live audit rules; `ausearch` queries `/var/log/audit/audit.log`.

```bash
sudo apt install auditd audispd-plugins   # Debian/Ubuntu
sudo dnf install audit                    # Fedora/RHEL
```

## Syntax

```bash
auditctl [options]
ausearch [options]
```

## Common Commands & Options

### `auditctl` (Rules & Status)

| Option | Description |
|--------|-------------|
| `-l` | List currently active audit rules |
| `-s` | Print audit daemon status (enabled, PID, lost events) |
| `-w path -p rwxa -k key` | Watch a file/directory for Read, Write, eXecute, or Attribute changes |
| `-a always,exit -F arch=b64 -S syscall -k key` | Audit a specific 64-bit syscall |
| `-D` | Delete all active rules (careful!) |

### `ausearch` (Querying Logs)

| Option | Description |
|--------|-------------|
| `-k key` | Filter events by the rule key (`-k`) |
| `-f path` | Search events touching a specific file path |
| `-m TYPE` | Filter by message type (`EXECVE`, `SYSCALL`, `USER_LOGIN`, `AVC`) |
| `-ui UID` / `-ua AUID` | Filter by effective UID or original login Audit UID (`auid`) |
| `-ts recent` / `-ts today` | Filter by start time (`today`, `recent`, `14:00:00`) |
| `-i` | Interpret numeric UIDs, GIDs, syscalls, and exit codes into names |

## Key Use Cases

1. Track down **who** modified or deleted a critical config file (`/etc/sudoers`, `/etc/ssh/sshd_config`)
2. Debug silent permission denials (SELinux `AVC` denials or `EACCES` failures)
3. Record command executions (`execve`) for compliance or incident response
4. Inspect failed authentication and privilege escalation attempts

## Safety

- Overly broad syscall rules (such as auditing every `openat` or `read` without path/UID filters) can flood `/var/log/audit/audit.log`, fill disk, and degrade I/O performance.
- Rules added with `auditctl` are **ephemeral** until saved under `/etc/audit/rules.d/*.rules` and loaded with `augenrules --load`.
- Check `auditctl -s` for `lost` > `0`, which indicates the kernel audit backlog buffer overflowed.

## Examples with Explanations

### Check if auditing is active and list rules

```bash
sudo auditctl -s
sudo auditctl -l
```

Shows whether `enabled` is `1` (active), `2` (locked until reboot), or `0` (disabled), plus any active watch/syscall rules.

### Watch a sensitive file for writes and attribute changes

```bash
sudo auditctl -w /etc/ssh/sshd_config -p wa -k sshd_config_watch
```

Records every write (`w`) or permission/ownership change (`a`) to `/etc/ssh/sshd_config`, tagged with the searchable key `sshd_config_watch`.

### Query who changed the watched file (with human-readable names)

```bash
sudo ausearch -k sshd_config_watch -i
```

`-i` translates numeric IDs (`auid=1000` -> `alice`) and syscall numbers into readable names, showing both the login user (`auid`) and effective user (`euid=root` via `sudo`).

### Find recent SELinux AVC denials or failed syscalls

```bash
sudo ausearch -m AVC -ts recent
sudo ausearch -sc openat --success no -ts today -i | tail -n 30
```

Quickly surfaces denied operations without grepping raw multi-line audit records.

### Search all events for a specific file or command

```bash
sudo ausearch -f /etc/passwd -i
sudo ausearch -c useradd -ts today -i
```

### Persist rules across reboots

Create `/etc/audit/rules.d/99-custom.rules`:

```text
-w /etc/ssh/sshd_config -p wa -k sshd_config_watch
-w /etc/sudoers -p wa -k sudoers_watch
```

Then compile and load:

```bash
sudo augenrules --load
```

## Understanding Output

A decoded `ausearch -i` record groups related lines (`SYSCALL`, `CWD`, `PATH`, `PROCTITLE`) under one `msg=audit(timestamp:serial)`:

| Field | Meaning |
|-------|---------|
| `auid` | **Audit UID** — the original logged-in user (persists across `sudo`/`su`) |
| `uid` / `euid` | User and effective user executing the syscall |
| `comm` / `exe` | Process name and full binary path |
| `key` | Custom tag assigned via `-k` |
| `success=yes/no` | Whether the kernel syscall succeeded |

## Notes & Pitfalls

- `auid` is the single most valuable field in forensics because it tracks the real human user even after `sudo su -`.
- Raw `/var/log/audit/audit.log` uses multi-line records and hex-encoded `proctitle` strings—always prefer `ausearch -i` over `grep` on the raw file.
- Use `aureport` (bundled with `auditd`) for quick high-level summaries (`sudo aureport --failed`, `sudo aureport -f`).

## Related Commands

- `dmesg` — kernel ring buffer messages
- `journalctl` — systemd journal logs
- `logger` — write custom syslog entries from scripts
- `strace` — live per-process syscall tracing (interactive debugging vs. `auditd` background logging)

## Additional Resources

- `man auditctl`
- `man ausearch`
- `man audit.rules`
