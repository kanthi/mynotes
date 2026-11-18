# btrfs

## Overview

`btrfs` (B-tree Filesystem) is the user-space management tool for **Btrfs** filesystems—a modern Copy-on-Write (CoW) Linux filesystem with built-in subvolumes, atomic snapshots, transparent compression (`zstd`), data/metadata checksums, and multi-device RAID. It is the default root filesystem on Fedora Workstation and openSUSE.

## Syntax

```bash
btrfs <subcommand> [options] <path_or_device>
```

## Common Subcommands

| Subcommand | Description |
|------------|-------------|
| `filesystem df PATH` | Show true allocated vs. used space (metadata, data, system) |
| `filesystem usage PATH` | Detailed space, device allocation, and unallocated space breakdown |
| `subvolume list PATH` | List all subvolumes and snapshots |
| `subvolume snapshot SRC DST` | Create a writable (or `-r` read-only) instant snapshot |
| `subvolume delete PATH` | Delete a subvolume or snapshot |
| `scrub start PATH` | Verify data and metadata checksums on disk (`scrub status` to monitor) |
| `balance start PATH` | Rebalance data/metadata chunks across devices or reclaim empty block groups |
| `device stats PATH` | Check for read/write/corruption I/O errors per device |

## Key Use Cases

1. Take instant read-only snapshots before risky system upgrades or migrations
2. Inspect true free space when `df` looks confusing on a CoW filesystem
3. Run online checksum verification (`scrub`) to catch silent bit-rot
4. Check device error counters (`device stats`) early

## Safety

- Classic `df` can be misleading on Btrfs because space is split into data and metadata chunks—always trust `btrfs filesystem usage` when diagnosing "No space left on device".
- Deleting large files on a full Btrfs filesystem can fail if metadata chunks are 100% full; keep snapshots pruned and avoid letting disks hit 95%+ capacity.
- Avoid Btrfs RAID5/6 for critical production data; RAID1/RAID10 are battle-tested.

## Examples with Explanations

### Inspect real space and metadata allocation

```bash
sudo btrfs filesystem usage /
```

Shows `Device size`, `Device allocated`, `Device unallocated`, and `Free (estimated)` clearly.

### Create and list snapshots

```bash
sudo btrfs subvolume snapshot -r /@home /@snapshots/home-2026-11-18
sudo btrfs subvolume list /
```

`-r` creates an instant, zero-cost read-only snapshot; only future changes consume new blocks.

### Run an online integrity scrub and check device errors

```bash
sudo btrfs scrub start /
sudo btrfs scrub status /
sudo btrfs device stats /
```

Scrub reads every block and verifies its checksum in the background without unmounting.

### Reclaim unallocated chunks after deleting many files

```bash
sudo btrfs balance start -dusage=50 /
```

Compacts data block groups that are less than 50% full without rewriting the entire filesystem.

## Notes & Pitfalls

- Disable CoW (`chattr +C /path/to/dir`) on directories holding large, randomly updated database files or VM images if fragmentation hurts performance (note: `+C` also disables Btrfs checksums for those files).
- Use `compsize` (separate package) to measure how much space Btrfs `zstd` compression is actually saving.

## Related Commands

- `df` / `duf` — generic filesystem space tools
- `fsck` — note: Btrfs uses `btrfs check` (run only unmounted, and only after reading docs)
- `snapper` — automated snapshot manager built on `btrfs`

## Additional Resources

- `man btrfs`
- `man btrfs-subvolume`
