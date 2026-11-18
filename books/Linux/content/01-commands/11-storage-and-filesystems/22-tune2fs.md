# tune2fs

## Overview

`tune2fs` inspects and adjusts tunable superblock parameters on **ext2, ext3, and ext4** filesystems—including volume labels, UUIDs, reserved root block percentages, periodic `fsck` intervals, and filesystem feature flags. Always inspect current settings with `tune2fs -l` before modifying a volume.

## Syntax

```bash
sudo tune2fs [options] device
```

## Common Options

| Option | Description |
|--------|-------------|
| `-l` | List all superblock parameters and feature flags (read-only) |
| `-L label` | Set the filesystem volume label (max 16 chars) |
| `-U UUID\|random\|time` | Set or regenerate the filesystem UUID |
| `-m percent` | Set percentage of blocks reserved for `root` (default `5%`) |
| `-r blocks` | Set exact count of reserved blocks |
| `-c max-mount-counts` | Max mounts before forced `fsck` (`-1` or `0` disables) |
| `-i interval[d\|w\|m]` | Max time between `fsck` runs (`0` disables) |
| `-o mount-options` | Set default mount options embedded in the superblock |
| `-O [^]feature` | Enable or disable (`^`) ext* feature flags (careful) |

## Key Use Cases

1. Reclaim wasted `5%` root-reserved space on large pure-data disks (`-m 1` or `-m 0`)
2. Set human-friendly labels (`-L`) or regenerate duplicate UUIDs (`-U random`) after cloning disks
3. Inspect filesystem creation date, block size, and mount count (`-l`)
4. Disable annoying boot-time mount-count `fsck` delays on large data arrays (`-c -1 -i 0`)

## Safety

- Double-check the target device with `lsblk -f` and `blkid` before writing.
- Changing `-U` (UUID) on a root or `/etc/fstab`-mounted disk without updating `/etc/fstab` and bootloader configs will prevent the system from booting.
- Enabling newer `-O` feature flags (e.g., `metadata_csum`) can make a volume unmountable on older kernels.

## Examples with Explanations

### Inspect key superblock settings

```bash
sudo tune2fs -l /dev/vg0/root | grep -iE 'volume name|uuid|block count|reserved|mount count|features|created'
```

### Reclaim reserved space on a large data disk

```bash
sudo tune2fs -m 1 /dev/vg0/data
```

On a 10 TB data drive, lowering `-m` from the default `5%` to `1%` (or `0%` for non-root data volumes) immediately frees ~400 GB for normal users.

### Set a volume label and regenerate a cloned UUID

```bash
sudo tune2fs -L BACKUPS /dev/sdb1
sudo tune2fs -U random /dev/sdb1
sudo blkid /dev/sdb1
```

Essential after cloning a disk with `dd` so two partitions do not share the same UUID.

### Disable mount-count and time-based forced `fsck`

```bash
sudo tune2fs -c -1 -i 0 /dev/vg0/data
```

Note: the kernel still replays the journal on mount and checks dirty flags after unclean shutdowns.

### Enable `dir_index` or check default mount options

```bash
sudo tune2fs -o discard /dev/sdb1
```

## Notes & Pitfalls

- Keep the default `5%` (or at least `1–2%`) reserved on `/` (root filesystem) so `root` and system daemons can still log in and write logs if a normal user fills the disk.
- `tune2fs` only works on ext2/ext3/ext4; XFS uses `xfs_admin`, and Btrfs uses `btrfs filesystem label`.

## Related Commands

- `blkid` / `e2label` — inspect labels and UUIDs
- `resize2fs` — grow/shrink ext* filesystems
- `e2fsck` / `fsck` — check and repair ext* filesystems
- `dumpe2fs` — even more detailed superblock and block-group dump

## Additional Resources

- `man tune2fs`
