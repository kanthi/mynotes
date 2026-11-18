# restorecon

## Overview

`restorecon` resets **SELinux security contexts** (file labels) on files and directories to match the system's policy rules (`file_contexts`). It is the #1 fix when a service like `nginx` or `httpd` fails to read files you moved (`mv`) from your home directory, because `mv` preserves the source label (`user_home_t`) instead of assigning the destination directory's label (`httpd_sys_content_t`).

## Syntax

```bash
sudo restorecon [options] path...
```

## Common Options

| Option | Description |
|--------|-------------|
| `-v` | Verbose—print every file whose label was changed |
| `-R` | Recurse into subdirectories |
| `-n` | Dry-run—show what labels would change without modifying anything |
| `-F` | Force reset (even the user/role/range portion, not just the type) |
| `-e DIR` | Exclude a directory during recursive relabeling |
| `-p` | Show progress percentage (helpful on huge trees) |

## Key Use Cases

1. Fix `Permission denied` errors after moving files into `/var/www/html`, `/srv`, or `~/.ssh`
2. Dry-run a relabel (`-Rvn`) to see which files have drifted from policy
3. Apply newly added `semanage fcontext` rules to an existing custom data directory

## Safety

- Always preview large recursive relabels with `-n` (`restorecon -Rvn /path`) first so you don't unexpectedly alter custom labels.
- Remember that `restorecon` only knows about paths defined in policy or `semanage fcontext`—if you move web roots to `/opt/myapp`, register the path rule with `semanage fcontext` first.

## Examples with Explanations

### Dry-run before relabeling a web root

```bash
sudo restorecon -Rvn /var/www/html
```

Lists any mismatched labels without changing them.

### Relabel a web directory recursively and verify

```bash
sudo restorecon -Rv /var/www/html
ls -Z /var/www/html | head
```

### Fix `~/.ssh/authorized_keys` after manual copying

```bash
restorecon -Rv ~/.ssh
```

If `~/.ssh/authorized_keys` has the wrong SELinux type, `sshd` will silently reject key authentication.

### Register a custom directory rule, then apply with `restorecon`

```bash
sudo semanage fcontext -a -t httpd_sys_content_t '/srv/mysite(/.*)?'
sudo restorecon -Rv /srv/mysite
```

`semanage fcontext` persists the rule; `restorecon` applies it to existing files.

## Notes & Pitfalls

- `cp` creates new files inheriting the destination directory's default label, whereas `mv` within the same filesystem keeps the old label—use `cp` (or `restorecon` after `mv`) when moving files into confined service directories.
- On Debian/Ubuntu systems using AppArmor, `restorecon` is typically not installed or used.

## Related Commands

- `ls -Z` — view current SELinux file labels
- `chcon` — temporary manual label override (overwritten by `restorecon`)
- `semanage fcontext` — manage persistent path-to-label rules
- `getenforce` / `setenforce` — check/toggle SELinux mode

## Additional Resources

- `man restorecon`
- `man semanage-fcontext`
