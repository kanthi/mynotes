# wipefs

## Overview

`wipefs` probes and erases **filesystem, RAID, LVM, and partition-table magic signatures** from a block device without zeroing the entire disk. It is the cleanest way to prepare a reused drive that still carries stale LVM, `mdadm`, ZFS, or GPT metadata before repartitioning or reformatting. Part of `util-linux`.

## Syntax

```bash
sudo wipefs [options] device
```

## Common Options

| Option | Description |
|--------|-------------|
| *(no flags)* | Probe and print all detected signatures and byte offsets (read-only) |
| `-a`, `--all` | Erase all detected magic signatures on the device |
| `-o`, `--offset OFFSET` | Erase only the signature at a specific byte offset |
| `-n`, `--no-act` | Dry-run—show what would be erased without writing to disk |
| `-p`, `--parsable` | Output in machine-parsable colon-delimited format |
| `-t`, `--types LIST` | Limit probing/erasing to specific types (e.g., `-t ext4,xfs`) |
| `-f`, `--force` | Force erasure even if the filesystem is mounted or in use (**dangerous**) |

## Key Use Cases

1. Inspect hidden or leftover signatures on a disk (`wipefs /dev/sdb`) before reuse
2. Clear stale `LVM2_member` or `linux_raid_member` headers that block `mkfs` or `parted`
3. Selectively remove one conflicting signature at a specific offset (`-o`)
4. Dry-run a signature wipe (`-n -a`) before committing

## Safety

- Erasing signatures makes existing data **immediately unmountable**—always confirm the exact device with `lsblk -f` and `blkid` first.
- Never use `-f` on a mounted filesystem.
- When decommissioning an active LVM PV or `mdadm` RAID array, prefer proper teardown (`pvremove`, `mdadm --zero-superblock`) while the stack is intact.

## Examples with Explanations

### Probe a disk for all signatures (safe read-only check)

```bash
sudo wipefs /dev/sdb
sudo wipefs /dev/sdb1
```

Lists each signature's byte `OFFSET`, `TYPE` (e.g., `gpt`, `ext4`, `LVM2_member`), `UUID`, and `LABEL`.

### Preview what `-a` would remove (dry-run)

```bash
sudo wipefs -n -a /dev/sdb
```

### Erase all signatures and verify with `blkid`

```bash
sudo wipefs -a /dev/sdb
sudo blkid /dev/sdb          # returns nothing (exit code 2)
```

Removes every filesystem/RAID/partition magic string in milliseconds without writing gigabytes of zeros.

### Remove only one specific signature by offset

```bash
sudo wipefs -o 0x438 /dev/sdb1
```

Useful when a disk has overlapping signatures (for example, an old `ext4` superblock left inside a newer `xfs` partition).

### Full clean-slate workflow: wipe, partition, and format

```bash
sudo wipefs -a /dev/sdb
sudo parted -s /dev/sdb mklabel gpt
sudo parted -s /dev/sdb mkpart primary xfs 1MiB 100%
sudo mkfs.xfs /dev/sdb1
```

## Notes & Pitfalls

- `wipefs -a /dev/sdb` wipes signatures on the **whole-disk** device (like the GPT table), while old partition signatures inside `/dev/sdb1` may still linger if you recreate a partition at the exact same offset—wipe the partition first (`wipefs -a /dev/sdb*`) if needed.
- `wipefs` does **not** securely erase user data; use `blkdiscard`, `nvme sanitize`, or `shred` when decommissioning drives with sensitive data.

## Related Commands

- `blkid` — probe and print block device UUIDs and types
- `lsblk -f` — inspect filesystem and mount hierarchy
- `parted` / `fdisk` — create fresh partition tables after wiping
- `dd` — overwrite blocks with zeros (much slower than `wipefs`)

## Additional Resources

- `man wipefs`
