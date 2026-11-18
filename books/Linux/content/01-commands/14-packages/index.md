---
title: Intro
---

# Intro

Install and maintain software across Linux distributions and runtimes: Debian/Ubuntu (`apt`/`dpkg`), Fedora/RHEL (`dnf`/`rpm`/`yum`), Arch (`pacman`), Alpine (`apk`), openSUSE (`zypper`), plus sandboxed app formats (`snap`, `flatpak`).

## Commands in this part

| Command | Role |
|---------|------|
| `apt` | apt is the friendly package manager front-end on Debian and Ubuntu. |
| `dpkg` | The dpkg (Debian Package) command is a low-level package manager for Debian-based systems. |
| `dnf` | dnf (Dandified YUM) is the primary high-level package manager on Fedora, RHEL 8+, CentOS Stream, Rocky, AlmaLinux,… |
| `rpm` | The rpm (RPM Package Manager) command is a low-level package manager for RPM-based Linux distributions. |
| `yum` | yum (Yellowdog Updater Modified) is the classic RPM package manager front-end for older RHEL/CentOS/Scientific Linux… |
| `snap` | snap installs sandboxed Snap packages from the Snap Store. |
| `pacman` | pacman is the binary package manager for Arch Linux and its derivatives. |
| `apk` | apk (Alpine Package Keeper) is the fast, minimal package manager for Alpine Linux and containers. |
| `zypper` | zypper is the command-line package manager for openSUSE and SUSE Linux Enterprise. |
| `flatpak` | flatpak installs and sandboxes Linux desktop applications across distributions (typically from Flathub). |

## Quick Distro Cheat Sheet

| Task | Debian / Ubuntu | Fedora / RHEL | Arch | Alpine | openSUSE |
|------|-----------------|---------------|------|--------|----------|
| Refresh & upgrade | `apt update && apt upgrade` | `dnf upgrade` | `pacman -Syu` | `apk update && apk upgrade` | `zypper dup` (Tumbleweed) / `zypper up` (Leap) |
| Install package | `apt install PKG` | `dnf install PKG` | `pacman -Syu PKG` | `apk add --no-cache PKG` | `zypper install PKG` |
| Search package | `apt search TERM` | `dnf search TERM` | `pacman -Ss TERM` | `apk search TERM` | `zypper search TERM` |
| Remove + deps | `apt autoremove PKG` | `dnf remove PKG` | `pacman -Rns PKG` | `apk del PKG` | `zypper remove --clean-deps PKG` |
| File owner | `dpkg -S /path` | `rpm -qf /path` | `pacman -Qo /path` | `apk info -W /path` | `zypper search --provides /path` |

## Suggested starting points

1. Debian/Ubuntu day-to-day: `apt` (low-level: `dpkg`).
2. Fedora/RHEL: `dnf` (low-level: `rpm`; legacy: `yum`).
3. Arch Linux: `pacman`.
4. Containers / minimal VMs: `apk`.
5. openSUSE / SLES: `zypper`.
6. Sandboxed apps: `snap` (Ubuntu) or `flatpak` (desktop).

## Related parts

- Services and runtime — restart units after library upgrades
- Security — trust and third-party repos
- Files and paths — where binaries land (`which`, `dpkg -L`)

Continue with the individual command pages in this part.
