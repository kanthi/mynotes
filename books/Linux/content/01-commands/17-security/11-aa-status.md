# aa-status

## Overview

`aa-status` (from `apparmor-utils`) reports the current status of **AppArmor**—the path-based Mandatory Access Control (MAC) framework enabled by default on Ubuntu, Debian, and openSUSE (the counterpart to SELinux on Fedora/RHEL). It lists loaded security profiles and shows which processes are running in **enforce** mode vs. **complain** (log-only) mode.

## Syntax

```bash
sudo aa-status [options]
```

## Common Options

| Option | Description |
|--------|-------------|
| `--enabled` | Exit `0` if AppArmor is enabled, non-zero otherwise (ideal for scripts) |
| `--enforced` | Print count of profiles in enforce mode |
| `--complaining` | Print count of profiles in complain mode |
| `--json` / `--pretty-json` | Output profile and process status in JSON |
| `--verbose` | Detailed summary |

## Key Use Cases

1. Check whether AppArmor is blocking a service (`aa-status` + `journalctl -k | grep DENIED`)
2. Audit which daemons (`mysqld`, `cupsd`, `tcpdump`, `libvirtd`) have active MAC confinement
3. Switch a misbehaving profile to complain mode temporarily with `aa-complain` during debugging

## Examples with Explanations

### Check AppArmor status and active profiles

```bash
sudo aa-status
```

Shows total loaded profiles, how many are in `enforce` vs. `complain` mode, and unconfined processes that have a profile defined.

### Check for AppArmor denials in kernel logs

```bash
sudo dmesg -T | grep -i apparmor
# or via journalctl:
sudo journalctl -k --grep='apparmor="DENIED"' -n 20
```

Look for `operation="open"`, `name="..."`, and `requested_mask="..."` to see the exact file path AppArmor blocked.

### Temporarily set a profile to complain (audit-only) mode

```bash
sudo aa-complain /usr/sbin/tcpdump
# restore enforcement after fixing the profile:
sudo aa-enforce /usr/sbin/tcpdump
```

*(Requires `apparmor-utils` package.)*

## Notes & Pitfalls

- Unlike SELinux (which labels inodes via extended attributes), AppArmor matches **filesystem paths**—moving or symlinking a service's data directory outside its expected path (e.g., moving MySQL datadir to `/data/mysql`) is a common cause of silent `DENIED` errors.
- Profile files live in `/etc/apparmor.d/`, with local overrides cleanly placed in `/etc/apparmor.d/local/`.

## Related Commands

- `getenforce` / `setenforce` — SELinux equivalents on RHEL/Fedora
- `dmesg` / `journalctl` — inspect `apparmor="DENIED"` messages
- `ausearch` — query auditd logs (`-m AVC`)

## Additional Resources

- `man aa-status`
- `man apparmor`
