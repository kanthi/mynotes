# loginctl

## Overview

`loginctl` manages **systemd-logind** seats, login sessions, and user instances. It lets you see who is logged in (locally or via SSH), inspect session properties, cleanly terminate stale sessions, and toggle **lingering** (`enable-linger`)—which allows `systemctl --user` services, user timers, and rootless Podman containers to keep running after you log out.

## Syntax

```bash
loginctl [options] <command> [args...]
```

## Common Commands

| Command | Description |
|---------|-------------|
| `list-sessions` | List all active login sessions with ID, UID, user, seat, and TTY |
| `list-users` | List users with active sessions and their state (`active`, `lingering`, `closing`) |
| `session-status [ID]` | Show detailed status, leader PID, and cgroup processes for a session |
| `user-status [USER]` | Show a user's sessions, linger status, and running user services |
| `show-session ID -p PROP` | Print specific machine-readable session properties (`Remote`, `Type`, `State`) |
| `enable-linger [USER]` | Allow a user's `systemd --user` manager to start at boot and survive logout |
| `disable-linger [USER]` | Stop keeping a user's background manager alive after logout |
| `terminate-session ID` | End a session and kill all processes in its session scope |
| `lock-session [ID]` | Trigger screen lock on a graphical session |

## Key Use Cases

1. Enable `enable-linger` so rootless Podman containers or `systemctl --user` timers survive SSH logout
2. Inspect active SSH/console sessions and their process trees (`session-status`)
3. Terminate a hung or orphaned user session (`terminate-session`)
4. Script-friendly checks for local vs. remote (`ssh`) sessions (`show-session`)

## Safety

- `terminate-session` and `terminate-user` **kill all processes** in that session/user scope (including active SSH shells and unsaved editor buffers)—check `session-status` first.
- Enabling linger (`enable-linger`) lets a user run persistent background services across reboots; audit `/var/lib/systemd/linger/` on multi-tenant hosts.

## Examples with Explanations

### List active sessions and users

```bash
loginctl list-sessions
loginctl list-users
```

### Inspect a session's process tree and metadata

```bash
loginctl session-status
loginctl show-session "$XDG_SESSION_ID" -p Remote -p Type -p State
```

`Type` distinguishes `tty`, `x11`, `wayland`, and `unspecified`; `Remote=yes` indicates SSH.

### Enable lingering for rootless containers or user timers

```bash
sudo loginctl enable-linger deploy
loginctl user-status deploy | grep Linger
```

Without `enable-linger`, `systemd --user` exits as soon as the user's last SSH session closes, stopping user timers and containers.

### Check all users with lingering enabled

```bash
ls -la /var/lib/systemd/linger/
```

### Terminate a stale session cleanly

```bash
loginctl list-sessions
sudo loginctl terminate-session 42
```

## Notes & Pitfalls

- `enable-linger` creates an empty marker file in `/var/lib/systemd/linger/USER`.
- If `systemctl --user` fails with `Failed to connect to bus` after `su` or `sudo`, it's because `su`/`sudo` does not open a full PAM `logind` session with `XDG_RUNTIME_DIR` set—use `ssh user@localhost` or `machinectl shell user@`.
- Minimal containers without `systemd-logind` do not have `loginctl` sessions.

## Related Commands

- `systemctl` (`systemctl --user`) — manage per-user services and timers
- `who` / `w` / `last` — classic login reporting
- `machinectl` — open clean `logind` shells (`machinectl shell user@`)

## Additional Resources

- `man loginctl`
- `man logind.conf`
