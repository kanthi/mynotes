# resize2fs

## Overview

`resize2fs` grows or shrinks **ext2, ext3, and ext4** filesystems. Growing an ext4 filesystem **online** (while mounted) is routine after expanding an LVM Logical Volume (`lvextend`) or cloud partition (`growpart`). Shrinking is **offline-only** (unmounted) and requires a forced filesystem check first.

## Syntax

```bash
sudo resize2fs [options] device [size]
```

If `size` is omitted, `resize2fs` expands the filesystem to fill the entire underlying partition or Logical Volume.

## Common Options

| Option | Description |
|--------|-------------|
| `-p` | Display a completion progress bar |
| `-M` | Shrink the filesystem to the smallest possible size (offline only) |
| `-P` | Print the estimated minimum size of the filesystem without modifying it |
| `-f` | Force resize (overrides safety checks—rarely needed) |
| `-d flags` | Enable debugging flags |

## Key Use Cases

1. Grow an ext4 root or data filesystem online after `lvextend` or cloud disk expansion
2. Estimate the minimum shrinkable size before reclaiming space (`resize2fs -P`)
3. Shrink an unmounted ext4 filesystem before running `lvreduce`

## Safety

- **Order matters:**
  - **Growing:** expand the block device **first** (`lvextend`, `parted resizepart`, `growpart`), **then** run `resize2fs`.
  - **Shrinking:** unmount, run `e2fsck -f`, run `resize2fs` to the smaller size **first**, **then** shrink the LV/partition (never shrink the block device smaller than the filesystem!).
- Always back up important data before shrinking.
- XFS uses `xfs_growfs` (grow-only, takes a mountpoint); Btrfs uses `btrfs filesystem resize`.

## Examples with Explanations

### Grow an ext4 filesystem online to fill its Logical Volume

```bash
sudo lvextend -L +20G /dev/vg0/root
sudo resize2fs -p /dev/vg0/root
df -h /
```

*(Tip: `sudo lvextend -r -L +20G /dev/vg0/root` combines both steps automatically.)*

### Grow a cloud VM partition and filesystem

```bash
sudo growpart /dev/nvme0n1 1
sudo resize2fs /dev/nvme0n1p1
```

### Check minimum possible size before shrinking

```bash
sudo resize2fs -P /dev/vg0/data
```

Prints the minimum size in filesystem blocks (multiply by block size, usually 4096 bytes).

### Safely shrink an unmounted ext4 volume

```bash
sudo umount /mnt/data
sudo e2fsck -f /dev/vg0/data
sudo resize2fs -p /dev/vg0/data 30G
sudo lvreduce -L 30G /dev/vg0/data
sudo mount /dev/vg0/data /mnt/data
```

`e2fsck -f` is mandatory before `resize2fs` will allow an offline shrink.

## Notes & Pitfalls

- Growing online works on ext3/ext4, provided the filesystem was created with `resize_inode` (standard on all modern `mkfs.ext4` volumes); there is a practical online growth ceiling based on the initial `resize_inode` reservation if growing by 1000x.
- Watch unit prefixes: `G` (1000^3) vs. `GiB` (1024^3)—give yourself a tiny safety margin when shrinking an LV after `resize2fs`.

## Related Commands

- `lvextend` / `lvreduce` / `lvs` — LVM volume resizing
- `e2fsck` / `tune2fs` — ext* checking and tuning
- `xfs_growfs` — XFS online grow tool
- `df` / `findmnt` — verify new capacity

## Additional Resources

- `man resize2fs`
