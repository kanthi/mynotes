# pacman

## Overview

`pacman` (Package Manager) is the default binary package manager for **Arch Linux** and derivatives (Manjaro, EndeavourOS, CachyOS). It uses compressed `.pkg.tar.zst` archives, resolves dependencies automatically, and pairs a concise single-letter operation mode (`-S`, `-R`, `-Q`, `-F`) with modifier flags.

## Syntax

```bash
pacman <operation> [options] [targets...]
```

## Common Operations & Flags

| Flag | Description |
|------|-------------|
| `-S pkg` | Sync / install package(s) from remote repositories |
| `-Syu` | Refresh package databases (`-y`) and upgrade all installed packages (`-u`) |
| `-Ss query` | Search remote repositories |
| `-Si pkg` | Show detailed info for a remote package |
| `-Rns pkg` | Remove package (`-R`), unneeded deps (`-s`), and backup config files (`-n`) |
| `-Q` | Query locally installed packages (`-Qe` explicit, `-Qdt` orphans) |
| `-Ql pkg` | List all files installed by a local package |
| `-Qo /path` | Find which installed package owns a file |
| `-F /path` | Find which **remote** package provides a file (requires `pacman -Fy`) |
| `-Sc` / `-Scc` | Clean old cached packages (or wipe the entire cache) |

## Key Use Cases

1. Keep a rolling-release Arch system updated safely (`pacman -Syu`)
2. Install and cleanly remove packages without leaving orphan dependencies (`-Rns`)
3. Trace which package owns a binary or config file (`-Qo` / `-F`)
4. Audit explicitly installed packages vs. orphaned dependencies (`-Qe`, `-Qdt`)

## Safety

- **Never run partial upgrades** like `pacman -Sy pkg` without `-u`—on a rolling release, upgrading one library without upgrading the rest of the system can break binaries (soname mismatches). Always run `pacman -Syu` (or `pacman -Syu pkg`).
- Check Arch news (`https://archlinux.org/news/`) before major system upgrades if manual intervention is noted.
- Read `.pacnew` / `.pacsave` notices after upgrades and merge config changes promptly.

## Examples with Explanations

### Full system update and package install

```bash
sudo pacman -Syu
sudo pacman -Syu ripgrep fd jq
```

Refreshes repo indexes and upgrades the whole system before installing new tools.

### Search and inspect before installing

```bash
pacman -Ss '^nginx'
pacman -Si nginx
```

### Remove a package cleanly with unused dependencies

```bash
sudo pacman -Rns htop
```

`-Rns` removes `htop`, any dependencies no longer needed by other packages, and `/etc` backup files (`.pacsave`).

### Find and clean orphaned dependencies

```bash
pacman -Qdtq
sudo pacman -Rns $(pacman -Qdtq)
```

Lists packages installed as dependencies that are no longer required by anything, then removes them.

### Check file ownership (installed and remote)

```bash
pacman -Qo /usr/bin/curl
sudo pacman -Fy && pacman -F bin/rg
```

`-Qo` queries local packages; `-F` searches uninstalled repo packages (like `apt-file`).

### List explicitly installed packages (for backups or rebuilds)

```bash
pacman -Qqe > pkglist.txt
```

Saves a clean list of explicitly installed native packages (no version numbers) that you can reinstall later with `sudo pacman -S --needed - < pkglist.txt`.

## Notes & Pitfalls

- `pacman` only manages official Arch repositories; **AUR** (Arch User Repository) packages require `makepkg` or an AUR helper such as `paru` or `yay`.
- Enable parallel downloads and color in `/etc/pacman.conf` (`Color`, `ParallelDownloads = 5`).
- Keep at least 1–2 recent versions in `/var/cache/pacman/pkg/` (`paccache -r` from `pacman-contrib`) in case you need to downgrade (`pacman -U /var/cache/pacman/pkg/...`).

## Related Commands

- `apt` / `dnf` / `apk` / `zypper` — other distro package managers
- `makepkg` — build Arch packages from `PKGBUILD` recipes

## Additional Resources

- `man pacman`
- ArchWiki — pacman
