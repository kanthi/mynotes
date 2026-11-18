# namei

## Overview

`namei` walks a pathname **component by component**, displaying whether each segment is a directory, file, or symbolic link—and with `-l` / `-m`, showing the exact **owner, group, and permission bits** at every step. It is the fastest tool for diagnosing `Permission denied` or `Too many levels of symbolic links` errors buried halfway down a deep directory tree. Part of `util-linux`.

## Syntax

```bash
namei [options] pathname...
```

## Common Options

| Option | Description |
|--------|-------------|
| `-l`, `--long` | Long listing format: mode bits, owner, group, and symlink targets |
| `-m`, `--modes` | Display mode bits (`drwxr-xr-x`) for each component |
| `-o`, `--owners` | Show owner and group names for each component |
| `-x`, `--mountpoints` | Mark mount point boundaries (`D` instead of `d`) |
| `-n`, `--nosymlinks` | Do not follow symbolic links (show the link itself and stop) |
| `-v`, `--vertical` | Vertically align output columns |

## Key Use Cases

1. Debug `Permission denied` errors when a file is `644` yet a service (`nginx`, `www-data`, `postgres`) still cannot read it because a parent directory lacks `+x` (traverse) permission
2. Audit `~/.ssh/authorized_keys` ownership and modes across `/home`, `/home/user`, and `.ssh`
3. Trace nested symlinks (`/usr/bin/python` -> `python3` -> `python3.12`) and catch symlink loops
4. Spot where a path crosses onto a different filesystem mount (`-x`)

## Examples with Explanations

### Diagnose why a web server cannot read a file

```bash
namei -l /var/www/html/app/index.html
```

Checks every directory from `/` down to `index.html`. If `/var/www/html/app` is `drwx------ root root`, `www-data` cannot traverse it regardless of `index.html` permissions.

### Audit SSH directory permissions in one command

```bash
namei -l /home/alice/.ssh/authorized_keys
```

Confirms `/home/alice` and `.ssh` are not group/world-writable (which causes `sshd` to reject key login).

### Test path traversal as a service user (`www-data`)

```bash
sudo -u www-data namei -l /srv/releases/current/public/index.html
```

Runs `namei` as the target service account to see the exact component where permission fails.

### Trace mount boundaries along a path

```bash
namei -x /mnt/data/backups/daily.tar.zst
```

Components labeled `D` indicate mount point crossings.

### Show symlinks without following them

```bash
namei -n /usr/bin/vi
```

## Understanding Output

First character of each line in `namei`:

| Prefix | Meaning |
|--------|---------|
| `f:` | Path header being resolved |
| `d` | Directory |
| `D` | Directory that is also a mount point (shown with `-x`) |
| `l` | Symbolic link (followed on subsequent indented lines) |
| `-` | Regular file |
| `?` | Cannot stat/access (permission denied or broken symlink target) |

## Notes & Pitfalls

- Remember that `x` (execute/search) on a **directory** is required to traverse through it; `r` alone is not enough to open files inside.
- `namei -l` shows standard POSIX mode bits and ownership—if every mode bit looks right and access still fails, check POSIX ACLs (`getfacl`) and SELinux/AppArmor (`ls -Z`, `aa-status`).

## Related Commands

- `ls` (`ls -ld`) — inspect a single directory's metadata
- `realpath` / `readlink -f` — resolve the final canonical path
- `stat` — detailed inode metadata
- `getfacl` — inspect POSIX ACLs beyond standard mode bits

## Additional Resources

- `man namei`
