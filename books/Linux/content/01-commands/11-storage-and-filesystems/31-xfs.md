# xfs_info & xfs_repair

## Overview

`xfs_info`, `xfs_growfs`, and `xfs_repair` (from `xfsprogs`) are the core utilities for **XFS**—the high-performance 64-bit journaling filesystem used as the default on RHEL, Rocky Linux, AlmaLinux, and many high-throughput storage servers. Unlike `ext4`, XFS **cannot be shrunk**—it can only be grown online.

```bash
sudo apt install xfsprogs   # Debian/Ubuntu
sudo dnf install xfsprogs   # Fedora/RHEL (preinstalled on root XFS)
```

## Syntax

```bash
xfs_info <mountpoint_or_device>
xfs_growfs <mountpoint>
xfs_repair [options] <unmounted_device>
```

## Common Commands & Options

| Command / Option | Description |
|------------------|-------------|
| `xfs_info /mount` | Print XFS geometry: block size, AG count, inode size, reflink/crc flags |
| `xfs_growfs /mount` | Grow an **online, mounted** XFS filesystem to fill its underlying partition/LV |
| `xfs_repair -n /dev/sdX1` | Dry-run check (read-only, reports corruption without modifying disk) |
| `xfs_repair /dev/sdX1` | Repair an **unmounted** XFS filesystem |
| `xfs_admin -L label /dev/sdX1` | Set or view the filesystem label (unmounted) |
| `xfs_fsr /mount` | Online XFS filesystem defragmenter |

## Key Use Cases

1. Grow an XFS root or data volume online after extending an LVM LV or cloud EBS disk
2. Inspect whether `reflink=1` (fast CoW copies via `cp --reflink`) and `crc=1` are enabled
3. Safely check (`-n`) or repair an unmounted XFS filesystem after a power/storage outage

## Safety

- **XFS cannot be shrunk.** Plan partition/LV sizes conservatively and grow as needed.
- Never run `xfs_repair` on a **mounted** filesystem. Always `umount` first (or use `-n` for read-only inspection).
- If the XFS log has unplayed metadata after a crash, mount and unmount cleanly once first before running `xfs_repair`; avoid `-L` (zero log) unless `xfs_repair` explicitly instructs you to, as it can discard recent metadata changes.

## Examples with Explanations

### Inspect XFS geometry and features

```bash
xfs_info /
```

Shows `bsize` (block size), `agcount` (allocation groups), `crc=1`, and `reflink=1`.

### Grow an XFS volume online after extending an LV

```bash
sudo lvextend -L +20G /dev/vg_data/lv_app
sudo xfs_growfs /var/lib/app
```

Note that `xfs_growfs` takes the **mount point** (`/var/lib/app`), whereas `resize2fs` (for ext4) takes the block device.

### Dry-run integrity check on an unmounted device

```bash
sudo umount /mnt/data
sudo xfs_repair -n /dev/sdb1
```

Checks metadata consistency without writing anything to disk.

### Repair after confirming issues

```bash
sudo xfs_repair /dev/sdb1
```

## Notes & Pitfalls

- `fsck.xfs` is intentionally a no-op script on Linux—XFS replays its journal automatically at mount time, and real repairs use `xfs_repair`.
- Because `reflink=1` is enabled by default on modern `mkfs.xfs`, `cp --reflink=auto` creates instant zero-copy clones on XFS just like on Btrfs.

## Related Commands

- `mkfs` (`mkfs.xfs`) — create an XFS filesystem
- `resize2fs` — ext2/ext3/ext4 resize tool (takes device, supports shrinking)
- `fsck` — generic filesystem check wrapper
- `lvs` / `lvextend` — LVM logical volume management

## Additional Resources

- `man xfs_info`
- `man xfs_growfs`
- `man xfs_repair`
