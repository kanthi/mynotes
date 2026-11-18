# pvs

## Overview

`pvs` reports **LVM physical volumes** (PVs)—disks or partitions initialized with `pvcreate` that supply raw storage extents to Volume Groups (VGs). Alongside `vgs` and `lvs`, `pvs` gives you an instant, scriptable inventory of your LVM storage layer.

```bash
sudo apt install lvm2   # Debian/Ubuntu
sudo dnf install lvm2   # Fedora/RHEL
```

## Syntax

```bash
sudo pvs [options] [PV...]
```

## Common Options

| Option | Description |
|--------|-------------|
| `-v`, `--verbose` | Include PV UUIDs, device sizes, and extent counts |
| `-o FIELDS` | Select specific columns (`pv_name`, `vg_name`, `pv_size`, `pv_free`, `pv_used`, `pv_attr`) |
| `-o +FIELDS` | Append extra columns to the default output |
| `--units h\|g\|m` | Display sizes in human-readable (`h`), GiB (`g`), or MiB (`m`) units |
| `-a`, `--all` | Show block devices that are not yet initialized as PVs |
| `--reportformat json` | Emit structured JSON for automation |

## Key Use Cases

1. Check how much unallocated space remains on each physical disk (`pvs`)
2. Verify which Volume Group owns a specific partition (`/dev/sdb1`)
3. Audit PV UUIDs and available extents before expanding or migrating storage
4. Export LVM inventory to JSON for scripts

## Safety

- Never run `pvremove` on a PV that still belongs to a Volume Group or holds active extents—migrate data off first with `pvmove`, then `vgreduce`.
- Always verify device identity with `lsblk` before running `pvcreate` on a disk.

## Examples with Explanations

### Default summary and extended space columns

```bash
sudo pvs
sudo pvs -o +pv_used,pv_free,dev_size,pv_uuid
```

Shows how much of each PV is already allocated to LVs (`PUsed`) vs. free (`PFree`).

### Show sizes in GiB with custom columns

```bash
sudo pvs --units g -o pv_name,vg_name,pv_size,pv_free
```

### Initialize a new partition as a PV and verify

```bash
sudo pvcreate /dev/sdb1
sudo pvs
```

### Grow a PV after expanding the underlying cloud disk/partition

```bash
sudo pvresize /dev/sdb1
sudo pvs
```

Makes newly added disk capacity available to the Volume Group.

### Export PV inventory as JSON

```bash
sudo pvs --reportformat json | jq '.report[].pv[] | {pv_name, vg_name, pv_free}'
```

## Understanding Output

Default columns:

| Column | Meaning |
|--------|---------|
| `PV` | Block device path (e.g., `/dev/sdb1`) |
| `VG` | Volume Group name (blank if not yet added to a VG) |
| `Fmt` | Metadata format (`lvm2`) |
| `Attr` | Status flags (`a--` = allocatable, `m--` = missing device) |
| `PSize` | Usable physical volume size |
| `PFree` | Unallocated space available for new/extended LVs |

## Notes & Pitfalls

- Initializing whole disks (`/dev/sdb`) works, but using a single GPT partition (`/dev/sdb1` with `lvm` flag) is safer because other OSes and tools won't mistake the disk for unpartitioned free space.
- If `pvs` shows `[unknown]` or missing devices, a disk may be disconnected or multipath/LUKS may not be unlocked yet.

## Related Commands

- `pvcreate` / `pvresize` / `pvmove` / `pvremove` — PV lifecycle management
- `vgs` / `lvs` — Volume Group and Logical Volume reports
- `lsblk` — underlying block devices

## Additional Resources

- `man pvs`
- `man lvm`
