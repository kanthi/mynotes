# apk

## Overview

`apk` (Alpine Package Keeper) is the fast, minimal package manager for **Alpine Linux**, widely used in Docker/OCI containers and lightweight VMs. It uses `.apk` packages, tracks desired state in `/etc/apk/world`, and supports one-command ephemeral installs (`--no-cache`) that keep container image layers small.

## Syntax

```bash
apk <command> [options] [packages...]
```

## Common Commands

| Command | Description |
|---------|-------------|
| `apk update` | Fetch latest repository index |
| `apk upgrade` | Upgrade installed packages to latest versions |
| `apk add PKG…` | Install package(s) |
| `apk add --no-cache PKG` | Update index in memory, install, and leave no cache on disk |
| `apk del PKG…` | Remove package(s) and unneeded dependencies |
| `apk search PATTERN` | Search available packages |
| `apk info` | List installed packages (`-L` files, `-R` deps, `-s` size, `-W` file owner) |
| `apk fix` | Repair or reinstall broken packages |
| `apk cache clean` | Remove cached `.apk` files |

## Key Use Cases

1. Install runtime or build tools cleanly inside Dockerfiles without bloating image size
2. Create and remove temporary build dependencies using virtual packages (`--virtual`)
3. Inspect which package owns a file or how much space a package uses on minimal hosts

## Safety

- In containers, prefer `apk add --no-cache` instead of `apk update && apk add ... && rm -rf /var/cache/apk/*` so no stale index layer is left behind.
- Pinning versions in Alpine (`pkg=1.2.3-r0`) can break later builds when Alpine drops older point releases from upstream mirrors—only pin when necessary.

## Examples with Explanations

### Container-friendly install without leaving cache

```bash
apk add --no-cache curl jq ca-certificates
```

Updates the index in memory and discards it after installation.

### Temporary build dependencies with `--virtual`

```bash
apk add --no-cache --virtual .build-deps gcc musl-dev make \
  && make && make install \
  && apk del .build-deps
```

Groups build tools under `.build-deps` so a single `apk del .build-deps` strips all of them cleanly in one step.

### Search and inspect packages

```bash
apk search -v 'nginx*'
apk info -L nginx        # files installed by nginx
apk info -s nginx        # installed size
apk info -W /usr/bin/jq  # which package owns /usr/bin/jq
```

### Update and upgrade a running Alpine host

```bash
apk update && apk upgrade --available
```

### Simulate a change before applying

```bash
apk add --simulate postgresql
```

## Notes & Pitfalls

- Alpine uses **musl libc** and **BusyBox** rather than glibc and GNU coreutils—some binaries or flags behave differently than on Debian/RHEL.
- Community packages live in the `community` repository; check `/etc/apk/repositories` if a package is not found.
- `/etc/apk/world` is a plain text list of top-level requested packages—editing it and running `apk fix` syncs the system to match.

## Related Commands

- `apt` / `dnf` / `pacman` — Debian, RHEL, and Arch package managers
- `podman` / `docker` — container engines where `apk` is most common

## Additional Resources

- `man apk`
- Alpine Linux Wiki — Alpine Package Keeper
