# umask

## Overview

`umask` sets or displays the **file mode creation mask** for the current shell—the permission bits that are **masked off (removed)** whenever a new file or directory is created. It is a shell builtin (and kernel per-process attribute); on Ubuntu/Debian, interactive login shells typically default to `0002` (user-private groups) or `0022`.

## Syntax

```bash
umask [-p] [-S] [mode]
```

## Common Options

| Option | Description |
|--------|-------------|
| *(no args)* | Print the current mask in octal (e.g., `0022`) |
| `-S` | Print the mask in symbolic form showing **allowed** bits (e.g., `u=rwx,g=rx,o=rx`) |
| `-p` | Print as a reusable `umask 0022` command |
| `mode` | Set a new mask in octal (`027`, `077`) or symbolic (`u=rwx,go=`) form |

## Key Use Cases

1. Tighten default permissions on multi-user servers (`umask 027` or `077`)
2. Understand why newly created files default to `644` (`666 & ~022`) and directories to `755` (`777 & ~022`)
3. Safely create temporary secrets/tokens in scripts without a race condition before `chmod`
4. Configure service-specific masks in systemd units (`UMask=0027`)

## Safety

- `umask` only affects **newly created** files and directories—it does not change permissions on existing files (`chmod` does).
- When writing secrets (API keys, private keys) in a script, set `umask 077` **before** creating the file (or use `install -m 600`) so there is never a brief window where the file is world-readable.

## Examples with Explanations

### Inspect the current mask in octal and symbolic forms

```bash
umask
umask -S
```

Note that `umask` prints bits **removed** (`0022`), while `umask -S` prints bits **kept** (`u=rwx,g=rx,o=rx`).

### See how `umask` calculates default file and directory modes

```bash
umask 022
touch /tmp/testfile && mkdir /tmp/testdir
ls -ld /tmp/testfile /tmp/testdir
# file: 666 & ~022 = 644 (-rw-r--r--)
# dir:  777 & ~022 = 755 (drwxr-xr-x)
```

Files start from `666` (no execute bits by default); directories start from `777`.

### Enforce strict group-only or owner-only defaults

```bash
umask 027   # new files 640, dirs 750 (others get no access)
umask 077   # new files 600, dirs 700 (private to owner only)
```

### Safely write a secret file in a script without a race window

```bash
old_umask=$(umask)
umask 077
printf '%s\n' "$API_TOKEN" > "$HOME/.config/myapp/token"
umask "$old_umask"
```

### Persist `umask` for login shells vs. systemd services

```bash
# Interactive/login shells (~/.profile or /etc/profile.d/umask.sh):
umask 027

# Systemd service unit (/etc/systemd/system/myapp.service):
# [Service]
# UMask=0027
```

## Notes & Pitfalls

- `umask` is **not** a subtraction—it is a bitwise `AND NOT` (`base & ~umask`). For example, `666` with `umask 011` still yields `666` because files have no execute bits to strip.
- Systemd services, `cron` jobs, and `ssh` non-interactive commands often ignore `~/.bashrc`—set `UMask=` directly in the `.service` unit when daemons write files.
- Directory default ACLs (`setfacl -d`) override `umask` on directories that have them.

## Related Commands

- `chmod` / `chown` — change permissions/ownership on existing files
- `install` — copy or create files with explicit `-m` mode atomically
- `getfacl` / `setfacl` — manage POSIX default ACLs
- `stat` — inspect exact octal and symbolic modes

## Additional Resources

- `help umask` (in Bash)
- `man 2 umask`
