# mkswap, swapon & swapoff

## Overview

`mkswap`, `swapon`, and `swapoff` initialize, activate, and deactivate **Linux swap space** (either a swap partition or a swap file). Swap gives the kernel overflow space for anonymous memory pages under pressure and is required for hibernation.

## Syntax

```bash
mkswap [options] <device_or_file>
swapon [options] [device_or_file]
swapoff [options] <device_or_file | -a>
```

## Common Commands & Options

| Command / Option | Description |
|------------------|-------------|
| `swapon --show` | List active swap devices/files, sizes, usage, and priorities |
| `swapon -a` | Activate all swap entries in `/etc/fstab` |
| `swapon -p N target` | Activate swap with priority `N` (higher number = used first) |
| `swapoff -a` | Deactivate all active swap (pages are moved back into RAM) |
| `mkswap -L label target` | Format a partition or file as swap with an optional label |

## Key Use Cases

1. Add emergency swap space using a swap file without repartitioning
2. Inspect active swap usage and priority (`swapon --show`)
3. Disable swap safely (`swapoff -a`) before resizing a partition or running Kubernetes nodes that require swap off

## Safety

- **Never run `mkswap` on a data partition**—it overwrites the header and destroys filesystem metadata.
- Set swap file permissions to `0600` (`chmod 600 /swapfile`) so non-root users cannot read swapped memory pages.
- Before running `swapoff`, check `free -h` to confirm enough free RAM exists to hold the swapped-out pages, or the OOM killer may trigger.

## Examples with Explanations

### Inspect currently active swap

```bash
swapon --show
free -h
```

### Create and enable a 4 GiB swap file (ext4 / XFS)

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

To make it persistent across reboots, add to `/etc/fstab`:

```text
/swapfile none swap sw 0 0
```

*(Note: on **Btrfs**, create the file with `truncate -s 0 /swapfile && chattr +C /swapfile` before `dd`/`fallocate`, or use `btrfs filesystem mkswapfile --size 4g /swapfile` on newer `btrfs-progs`.)*

### Prioritize faster swap (e.g., zram over disk swap)

```bash
sudo swapon -p 100 /dev/zram0
sudo swapon -p 10 /swapfile
```

The kernel fills higher-priority swap (`100`) first before spilling to lower-priority disk swap (`10`).

### Disable swap cleanly

```bash
sudo swapoff /swapfile
```

## Notes & Pitfalls

- `fallocate` can create files with unwritten extents that older kernels reject for swap; if `swapon` complains about holes, create the swap file with `dd if=/dev/zero of=/swapfile bs=1M count=4096 status=progress`.
- Tune how aggressively the kernel swaps via `vm.swappiness` (`sysctl vm.swappiness`).

## Related Commands

- `free` — summary of RAM and swap usage
- `sysctl` — tune `vm.swappiness`
- `dd` / `fallocate` — allocate swap files

## Additional Resources

- `man swapon`
- `man mkswap`
