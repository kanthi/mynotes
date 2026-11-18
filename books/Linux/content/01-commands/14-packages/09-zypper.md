# zypper

## Overview

`zypper` is the command-line package manager for **openSUSE** (Tumbleweed, Leap) and **SUSE Linux Enterprise (SLES)**. Built on `libzypp`, it handles RPM repositories, patches, distribution upgrades (`dup`), and integrates with `snapper` on Btrfs root filesystems to create automatic pre/post-upgrade snapshots.

## Syntax

```bash
zypper [global-options] <command> [command-options] [args...]
```

## Common Commands

| Command | Alias | Description |
|---------|-------|-------------|
| `refresh` | `ref` | Refresh repository metadata |
| `update` | `up` | Update installed packages (Leap / SLES) |
| `dist-upgrade` | `dup` | Full distribution upgrade (required on rolling Tumbleweed) |
| `install PKG` | `in` | Install packages or local `.rpm` files |
| `remove PKG` | `rm` | Remove packages (`-u` / `--clean-deps` removes unused deps) |
| `search TERM` | `se` | Search packages (`-i` installed only, `--provides` by capability/file) |
| `info PKG` | `if` | Show package metadata, version, and repo |
| `list-patches` | `lp` | List applicable security/bugfix patches |
| `patch` | — | Apply official security/bugfix patches |
| `ps` | — | List running processes still using deleted/upgraded libraries |

## Key Use Cases

1. Maintain openSUSE Leap/SLES servers with targeted security patches (`zypper patch`)
2. Upgrade rolling-release openSUSE Tumbleweed safely (`zypper dup`)
3. Find which running daemons need a restart after library updates (`zypper ps`)
4. Lock critical package versions (`zypper addlock`)

## Safety

- On **openSUSE Tumbleweed**, always use `zypper dup` rather than `zypper up` so vendor/repo transitions and removals resolve cleanly.
- Use `-D` / `--dry-run` to preview large transactions before committing.
- On Btrfs systems with `snapper`, failed upgrades can be rolled back from the boot menu or `snapper rollback`.

## Examples with Explanations

### Refresh and update (Leap/SLES vs. Tumbleweed)

```bash
sudo zypper refresh
sudo zypper update          # openSUSE Leap / SLES
sudo zypper dup             # openSUSE Tumbleweed (rolling)
```

### Apply security patches only

```bash
sudo zypper list-patches --category security
sudo zypper patch --category security
```

### Install and remove with dependency cleanup

```bash
sudo zypper install -y ripgrep jq
sudo zypper remove --clean-deps ripgrep
```

### Search by package name or file provided

```bash
zypper search nginx
zypper search --provides /usr/bin/curl
zypper info nginx
```

### Check which processes need restarting after updates

```bash
sudo zypper ps -s
```

Lists services holding old shared libraries in memory (similar to `needrestart` on Debian/Ubuntu).

### Lock a package version

```bash
sudo zypper addlock postgresql16
zypper locks
sudo zypper removelock postgresql16
```

### Non-interactive scripting

```bash
sudo zypper --non-interactive install --no-recommends git curl
```

`--no-recommends` skips optional recommended packages to keep servers lean.

## Notes & Pitfalls

- `zypper` installs **recommended** packages by default (not just hard dependencies); pass `--no-recommends` on minimal servers.
- `zypper` supports concise two-letter aliases (`in`, `rm`, `se`, `up`, `dup`, `ref`, `lr`) for interactive speed.

## Related Commands

- `rpm` — low-level RPM queries and verification
- `dnf` — Fedora/RHEL RPM front-end
- `snapper` — Btrfs snapshot management on openSUSE

## Additional Resources

- `man zypper`
- openSUSE SDB — Zypper usage
