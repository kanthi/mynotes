# vgs

## Overview

`vgs` reports **LVM volume groups** (VGs)—named storage pools built from one or more Physical Volumes (PVs). Checking `vgs` tells you how many PVs and LVs are in each pool and, most importantly, how much free space (`VFree`) is available to create or extend Logical Volumes.

## Syntax

```bash
sudo vgs [options] [VG...]
```

## Common Options

| Option | Description |
|--------|-------------|
| `-v`, `--verbose` | Show VG UUID, extent size, and max LV/PV limits |
| `-o FIELDS` / `-o +FIELDS` | Choose or append columns (`vg_extent_size`, `vg_free_count`, `pv_count`, `lv_count`) |
| `--units h\|g\|m` | Format sizes in human-readable, GiB, or MiB units |
| `--reportformat json` | Output machine-readable JSON |
| `--noheadings` | Omit column headers (handy for shell scripts) |

## Key Use Cases

1. Check free pool capacity (`VFree`) before running `lvcreate` or `lvextend`
2. Inspect how many PVs and LVs belong to each Volume Group
3. Verify Physical Extent (PE) size and free extent counts (`vg_free_count`)
4. Combine with `vgextend` when adding a new disk to a full pool

## Safety

- Deleting a VG (`vgremove`) destroys metadata for all LVs inside it—always unmount and remove LVs first.
- A VG with missing PVs (`Attr` shows `p` for partial) should be inspected carefully before making metadata changes.

## Examples with Explanations

### List all Volume Groups and free space

```bash
sudo vgs
sudo vgs --units g -o vg_name,pv_count,lv_count,vg_size,vg_free
```

### Inspect extent size and free extent count

```bash
sudo vgs -o vg_name,vg_extent_size,vg_extent_count,vg_free_count
```

Useful when extending an LV by exact extents (`lvextend -l +100%FREE`).

### Create a Volume Group from two PVs

```bash
sudo vgcreate vg_data /dev/sdb1 /dev/sdc1
sudo vgs
```

### Add a new disk to an existing full Volume Group

```bash
sudo pvcreate /dev/sdd1
sudo vgextend vg_data /dev/sdd1
sudo vgs vg_data
```

### Script-friendly free space check

```bash
sudo vgs --noheadings --units g -o vg_free vg_data
```

## Understanding Output

Default columns:

| Column | Meaning |
|--------|---------|
| `VG` | Volume Group name |
| `#PV` / `#LV` | Number of Physical Volumes and Logical Volumes in the group |
| `#SN` | Number of snapshots |
| `Attr` | Attributes (`wz--n-`: writeable, resizable, normal allocation) |
| `VSize` | Total pooled capacity |
| `VFree` | Unallocated space ready for `lvcreate` or `lvextend` |

## Notes & Pitfalls

- Physical Extent (PE) size defaults to `4.00m` and is fixed at `vgcreate` time.
- If `VFree` is `0`, you can either add a new disk (`vgextend`) or shrink an oversized LV (ext4 only—XFS cannot shrink) to free extents.

## Related Commands

- `vgcreate` / `vgextend` / `vgreduce` / `vgremove` — VG lifecycle management
- `pvs` / `lvs` — PV and LV reports
- `lvmdiskscan` — scan all disks for LVM suitability

## Additional Resources

- `man vgs`
- `man vgcreate`
