# flatpak

## Overview

`flatpak` is a distro-agnostic package manager and sandboxing runtime for **Linux desktop applications**. Unlike `apt`, `dnf`, or `pacman`—which install system libraries globally—Flatpak bundles app dependencies into shared runtimes and isolates each app's filesystem, network, and device permissions. Most apps come from **Flathub**.

## Syntax

```bash
flatpak <command> [options] [app-id...]
```

## Common Commands

| Command | Description |
|---------|-------------|
| `remote-add` | Add a repository (typically Flathub) |
| `search QUERY` | Search available applications in configured remotes |
| `install [REMOTE] ID` | Install an application or runtime |
| `update` | Update all installed Flatpak apps and runtimes |
| `list` | List installed apps and runtimes (`--app` skips runtimes) |
| `uninstall ID` | Remove an application (`--unused` cleans orphaned runtimes) |
| `run ID` | Launch a Flatpak app from the terminal |
| `info ID` | Show version, origin, commit, and permissions |
| `override` | Grant or revoke sandbox permissions per app (or globally with `--user`) |

## Key Use Cases

1. Install up-to-date desktop apps on stable/LTS distros (Debian, Ubuntu LTS, RHEL, immutable distros) without PPAs
2. Run proprietary or untrusted desktop apps inside a restricted sandbox
3. Inspect and tighten an app's filesystem, network, or hardware permissions

## Safety

- Flatpak apps are sandboxed by default, but some apps request broad permissions (like `--filesystem=host` or `--socket=x11`); audit them with `flatpak info --show-permissions`.
- User-level installs (`--user`) stay inside `~/.local/share/flatpak` and do not require `sudo`.

## Examples with Explanations

### Add Flathub and install an app

```bash
flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
flatpak search obsidian
flatpak install flathub md.obsidian.Obsidian
```

### List installed apps and update everything

```bash
flatpak list --app
flatpak update -y
```

`--app` filters out shared runtimes and extensions so you only see applications.

### Run an app and inspect its sandbox permissions

```bash
flatpak run md.obsidian.Obsidian
flatpak info --show-permissions md.obsidian.Obsidian
```

### Override permissions (e.g., restrict or grant folder access)

```bash
flatpak override --user --nofilesystem=home md.obsidian.Obsidian
flatpak override --user --filesystem=~/Documents/Notes md.obsidian.Obsidian
flatpak override --user --show md.obsidian.Obsidian
```

Revokes broad home directory access and grants access only to `~/Documents/Notes`.

### Clean up unused runtimes and old versions

```bash
flatpak uninstall --unused
```

Removes old runtime versions no longer referenced by any installed app—this often reclaims gigabytes of disk space.

## Notes & Pitfalls

- Flatpak uses reverse-DNS application IDs (`org.mozilla.firefox`, `com.visualstudio.code`), though `flatpak install` accepts partial matches interactively.
- Initial installs can be large because they pull a base runtime (e.g., `org.freedesktop.Platform` or `org.gnome.Platform`), which subsequent apps share.
- On servers or headless containers, stick to native distro packages (`apt`, `dnf`, `apk`) rather than `flatpak`.

## Related Commands

- `snap` — Ubuntu's sandboxed package format (desktop and server)
- `apt` / `dnf` / `pacman` — native system package managers

## Additional Resources

- `man flatpak`
- Flathub — `https://flathub.org/`
