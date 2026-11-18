# getenforce

## Overview

`getenforce` prints the current **SELinux** enforcement mode: `Enforcing`, `Permissive`, or `Disabled`. It is the fastest first check on Fedora, RHEL, Rocky Linux, and AlmaLinux when a service fails with unexpected `Permission denied` errors even though standard `chmod`/`chown` permissions look right. (Ubuntu/Debian typically use AppArmor via `aa-status` instead.)

```bash
sudo dnf install policycoreutils   # Fedora/RHEL (usually preinstalled)
```

## Syntax

```bash
getenforce
```

## Key Use Cases

1. Quickly verify whether SELinux is actively blocking (`Enforcing`) or only logging (`Permissive`)
2. Confirm whether `setenforce 0` or `setenforce 1` took effect during incident triage
3. Check host MAC posture in automation scripts

## Examples with Explanations

### Check the active SELinux mode

```bash
getenforce
# Enforcing | Permissive | Disabled
```

### Inspect full SELinux policy and boolean status

```bash
sestatus
getsebool -a | grep httpd
```

`sestatus` adds the active policy name (`targeted`), mount point, and configured boot mode.

### Pair with `ausearch` when `Enforcing` blocks an app

```bash
getenforce
sudo ausearch -m AVC -ts recent
```

If `getenforce` reports `Enforcing`, checking recent `AVC` denials confirms whether SELinux blocked the action.

### Check configured boot-time mode

```bash
grep '^SELINUX=' /etc/selinux/config
```

## Notes & Pitfalls

- If `getenforce` returns `Disabled`, SELinux cannot be turned back on with `setenforce 1`—it requires setting `SELINUX=enforcing` (or `permissive`) in `/etc/selinux/config` and rebooting (plus a filesystem relabel).
- Inside unprivileged containers, `getenforce` may report `Disabled` even when the host is `Enforcing`.

## Related Commands

- `setenforce` — temporarily switch between `Enforcing` and `Permissive`
- `restorecon` — restore default SELinux file labels
- `ausearch` — query SELinux `AVC` denial logs
- `aa-status` — AppArmor status on Debian, Ubuntu, and openSUSE

## Additional Resources

- `man getenforce`
- `man selinux`
