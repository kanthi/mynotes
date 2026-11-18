# setenforce

## Overview

`setenforce` switches **SELinux** at runtime between **Enforcing** (`1` / `Enforcing`) and **Permissive** (`0` / `Permissive`). In Permissive mode, SELinux logs every policy violation as an `AVC` denial without blocking the operation—making `setenforce 0` the standard 30-second diagnostic test to prove whether SELinux is causing an application failure.

## Syntax

```bash
sudo setenforce 1   # or Enforcing
sudo setenforce 0   # or Permissive
```

## Common Arguments

| Argument | Effect |
|----------|--------|
| `1` / `Enforcing` | Actively block and log policy violations |
| `0` / `Permissive` | Allow operations, but still log `AVC` denials to audit/journal |

## Key Use Cases

1. Isolate whether a service failure is caused by SELinux vs. standard DAC permissions (`chmod`/`chown`)
2. Capture a clean set of `AVC` denials in Permissive mode so you can fix labels (`restorecon`) or booleans (`setsebool`)
3. Re-enable enforcement (`setenforce 1`) immediately after reproducing and fixing the issue

## Safety

- **Never leave production servers in Permissive mode permanently**—it disables Mandatory Access Control enforcement across the entire host.
- `setenforce` is **non-persistent** (resets to `/etc/selinux/config` on reboot).
- Fix the root cause (wrong file label with `restorecon`, missing boolean with `setsebool -P`, or custom port with `semanage port`) rather than disabling SELinux.

## Examples with Explanations

### Standard 3-step SELinux triage workflow

```bash
getenforce                          # 1. Confirm it is Enforcing
sudo setenforce 0                   # 2. Switch to Permissive briefly
# Reproduce the failing action, then inspect denials:
sudo ausearch -m AVC -ts recent
sudo setenforce 1                   # 3. Restore Enforcing immediately
```

### Confirm mode change in scripts or logs

```bash
sudo setenforce 0 && getenforce
```

### Persist mode across reboots (config file, not `setenforce`)

```bash
sudo sed -i 's/^SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config
```

## Notes & Pitfalls

- `setenforce` cannot switch SELinux from `Disabled` to `Enforcing`—if SELinux was disabled at boot, you must edit `/etc/selinux/config` and reboot.
- Use `semanage permissive -a httpd_t` when you only need **one** domain in permissive mode instead of lowering the whole system with `setenforce 0`.

## Related Commands

- `getenforce` / `sestatus` — check current and boot-time SELinux mode
- `restorecon` — fix mislabeled files
- `ausearch` / `audit2allow` — inspect `AVC` denials and policy fixes
- `aa-complain` — AppArmor's per-profile permissive equivalent on Ubuntu/Debian

## Additional Resources

- `man setenforce`
