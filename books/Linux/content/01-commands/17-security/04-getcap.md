# getcap / setcap

## Overview

Linux **file capabilities** split traditional `root` power into fine-grained privileges (such as `CAP_NET_BIND_SERVICE` to bind ports below 1024, or `CAP_NET_RAW` for raw sockets) attached directly to an executable via extended attributes (`xattr`). `getcap` inspects capabilities on files; `setcap` assigns or removes them—offering a safer alternative to `setuid` root binaries.

```bash
sudo apt install libcap2-bin   # Debian/Ubuntu
sudo dnf install libcap        # Fedora/RHEL
```

## Syntax

```bash
getcap [options] path...
sudo setcap [-q] [-v] <cap_spec> path
sudo setcap -r path
```

## Common Options & Capability Flags

| Option / Flag | Description |
|---------------|-------------|
| `getcap -r DIR` | Recursively scan a directory for binaries with capabilities |
| `setcap 'cap_...=+ep' FILE` | Assign Effective (`e`) and Permitted (`p`) capability |
| `setcap -r FILE` | Remove all file capabilities from `FILE` |
| `setcap -v 'cap_...' FILE` | Verify that `FILE` has the exact specified capability |
| `cap_net_bind_service` | Bind privileged TCP/UDP ports (`< 1024`) |
| `cap_net_raw` / `cap_net_admin` | Use raw sockets (`ping`, `tcpdump`) or configure network interfaces |
| `cap_dac_read_search` | Bypass file read permission checks (backup agents) |

## Key Use Cases

1. Let a non-root web server or binary bind port `80`/`443` (`cap_net_bind_service=+ep`)
2. Audit all capability-enabled binaries on a host (`getcap -r /`) during security reviews
3. Remove unnecessary capabilities (`setcap -r`) when hardening a system

## Safety

- Some capabilities (like `cap_sys_admin`, `cap_dac_override`, or `cap_setuid`) are effectively **full root**—do not grant them lightly.
- Package upgrades (`apt upgrade`, `dnf upgrade`) overwrite the binary and **wipe custom `setcap` attributes**; for systemd services, prefer `AmbientCapabilities=` in the unit file instead.
- Never grant capabilities to interpreted scripts (`python`, `bash` wrappers)—the kernel ignores file capabilities on `#!` scripts, and granting them to `/usr/bin/python3` itself grants them to *every* Python script on the host.

## Examples with Explanations

### Inspect capabilities on a binary or system-wide

```bash
getcap /usr/bin/ping
sudo getcap -r /usr/bin /usr/local/bin 2>/dev/null
```

### Allow a Go/Rust binary to bind port 443 without running as root

```bash
sudo setcap 'cap_net_bind_service=+ep' /usr/local/bin/caddy
getcap /usr/local/bin/caddy
```

### Allow non-root `tcpdump` capture (for a dedicated `pcap` group)

```bash
sudo chgrp pcap /usr/bin/tcpdump
sudo chmod 750 /usr/bin/tcpdump
sudo setcap 'cap_net_raw,cap_net_admin=eip' /usr/bin/tcpdump
```

### Remove all capabilities from a file

```bash
sudo setcap -r /usr/local/bin/caddy
```

### Preferred systemd alternative (survives package upgrades)

```ini
[Service]
User=caddy
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
```

## Notes & Pitfalls

- Capabilities are stored in `security.capability` extended attributes (`xattr`), so the filesystem must support xattrs (ext4, XFS, Btrfs do; FAT/some network mounts do not).
- Plain `cp` or `tar` without `--xattrs` drops file capabilities when copying binaries.

## Related Commands

- `capsh` — inspect and test process capability sets
- `systemctl` — configure `AmbientCapabilities=` on services
- `sudo` — user-level command elevation
- `getfacl` — filesystem ACLs

## Additional Resources

- `man getcap` / `man setcap`
- `man 7 capabilities`
