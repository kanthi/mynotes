# lvs

## Overview

`lvs` reports **LVM logical volumes** (LVs)—the virtual block devices (`/dev/VG/LV` or `/dev/mapper/VG-LV`) that you format with a filesystem and mount. Use `lvs` to inspect volume sizes, paths, thin-pool usage, snapshots, and volume state before resizing or removing storage.

## Syntax

```bash
sudo lvs [options] [LV...]
```

## Common Options

| Option | Description |
|--------|-------------|
| `-a`, `--all` | Include hidden/internal LVs (thin pool metadata, RAID subvolumes) |
| `-o FIELDS` / `-o +FIELDS` | Select or append columns (`lv_path`, `lv_dm_path`, `data_percent`, `origin`) |
| `--units h\|g\|m` | Display sizes in human-readable, GiB, or MiB units |
| `--segments` | Show underlying PV segment layout for each LV |
| `--reportformat json` | Emit structured JSON |

## Key Use Cases

1. Check exact LV sizes and `/dev/VG/LV` paths before `lvextend` or `resize2fs`
2. Monitor **snapshot** and **thin-pool** fill percentages (`Data%`, `Snap%`) before they hit 100%
3. Inspect RAID/mirror sync progress (`Cpy%Sync`)
4. Trace which physical disk holds which LV (`lvs -o +devices`)

## Safety

- If a classic thick **snapshot** reaches `100%` usage, it becomes **invalid** and is automatically dropped by the kernel—monitor `Snap%` closely.
- Always resize the LV (`lvextend`) **before** growing the filesystem (`resize2fs` / `xfs_growfs`), and shrink the filesystem **before** shrinking the LV (`lvreduce`).

## Examples with Explanations

### List LVs with full device paths and backing PVs

```bash
sudo lvs
sudo lvs -o lv_name,vg_name,lv_size,lv_path,devices,lv_attr
```

### Monitor thin pools and snapshots (`Data%`)

```bash
sudo lvs -a -o lv_name,vg_name,lv_size,pool_lv,data_percent,metadata_percent
```

Shows how full a thin pool or snapshot is so you can extend it before writes fail.

### Create, format, and mount a new Logical Volume

```bash
sudo lvcreate -n data -L 100G vg_data
sudo mkfs.ext4 /dev/vg_data/data
sudo mkdir -p /mnt/data
sudo mount /dev/vg_data/data /mnt/data
```

### Extend an LV and its filesystem online in one command

```bash
sudo lvextend -r -L +50G /dev/vg_data/data
```

Passing `-r` (`--resizefs`) automatically invokes `resize2fs` or `xfs_growfs` right after extending the LV—much safer than doing it manually.

### Create a snapshot before a risky upgrade

```bash
sudo lvcreate -s -n data_snap -L 10G /dev/vg_data/data
sudo lvs
```

## Understanding Output

The 6-character `Attr` (`lv_attr`) field summarizes volume state:

| Position | Common Values | Meaning |
|----------|---------------|---------|
| 1 (Type) | `-` normal, `s` snapshot, `t` thin pool, `V` thin volume, `r` RAID | Volume type |
| 2 (Perms) | `w` writable, `r` read-only | Access mode |
| 5 (State) | `a` active, `s` suspended, `I` invalid snapshot | Activation state |
| 6 (Open) | `o` open/mounted, `-` closed | Whether the device is currently open |

## Notes & Pitfalls

- Mount by **filesystem UUID** (`blkid`) in `/etc/fstab` or `/dev/mapper/VG-LV` consistently; both `/dev/VG/LV` and `/dev/mapper/VG-LV` are symlinks to the same `/dev/dm-N` device.
- Use `lvextend -l +100%FREE` to claim all remaining space in the Volume Group without calculating exact gigabytes.

## Related Commands

- `lvcreate` / `lvextend` / `lvreduce` / `lvremove` — LV lifecycle management
- `pvs` / `vgs` — PV and VG reports
- `resize2fs` / `xfs_growfs` — filesystem resizing
- `findmnt` — verify mounted LV paths

## Additional Resources

- `man lvs`
- `man lvcreate`
- `man lvextend`
