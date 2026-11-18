---
title: Intro
---

# Intro

Block devices, partitions, filesystems, mounts, LVM, encryption, integrity checks, and space usage. Destructive tools (`mkfs`, `dd`, `wipefs`, `cryptsetup luksFormat`, `mkswap`) always need identity double-checks.

## Commands in this part

| Command | Role |
|---------|------|
| `lsblk` | lsblk lists block devices (disks, partitions, LVM logical volumes, loop devices, ROM) as a tree. |
| `blkid` | blkid probes block devices for filesystem, RAID, and crypto signatures and prints tags such as TYPE, UUID, and LABEL. |
| `findmnt` | findmnt lists or searches mounted filesystems in tree or table form. |
| `fdisk` | fdisk creates and edits DOS/MBR and GPT partition tables on block devices. |
| `mkfs` | mkfs builds a filesystem on a block device (partition, LV, loop file). |
| `fsck` | fsck checks and optionally repairs filesystems. |
| `mount` | mount attaches a filesystem (block device, network share, bind path, tmpfs, …) onto a directory in the single Linux… |
| `umount` | umount detaches a mounted filesystem from the directory tree. |
| `dd` | dd copies and converts data at the block level. |
| `sync` | sync flushes filesystem buffers: dirty page cache and pending metadata are written to storage. |
| `losetup` | losetup associates regular files with loop devices (/dev/loopN) so the kernel can treat an image file like a block… |
| `duf` | duf is a modern, colorful disk usage / free space viewer — a friendlier alternative to df -h. |
| `dust` | dust is a modern du alternative that shows disk usage as a readable tree/bars, helping you find large directories… |
| `parted` | parted creates and modifies partition tables (GPT/MSDOS) on block devices. |
| `wipefs` | wipefs probes and can erase filesystem, RAID, or partition-table signatures from a device without zeroing the entire… |
| `smartctl` | smartctl (from smartmontools) queries disk S.M.A.R.T. |
| `pvs` | pvs reports LVM physical volumes (PVs) — disks or partitions initialized with pvcreate and used by volume groups. |
| `vgs` | vgs reports LVM volume groups (VGs): how physical volumes are pooled and how much space is free for logical volumes. |
| `lvs` | lvs reports LVM logical volumes (LVs) — the block devices you usually format and mount (/dev/VG/LV or… |
| `cryptsetup` | cryptsetup manages LUKS (and other) disk encryption: format, open, close, and resize encrypted block devices. |
| `resize2fs` | resize2fs grows or shrinks ext2/ext3/ext4 filesystems. |
| `tune2fs` | tune2fs adjusts tunable parameters on ext2/ext3/ext4 filesystems: labels, UUID, mount-count checks, reserved blocks,… |
| `ncdu` | ncdu (NCurses Disk Usage) is an interactive du for finding what consumes space. |
| `RWF_DONTCACHE` | RWF_DONTCACHE is the preadv2/pwritev2/io_uring flag for uncached buffered I/O: bulk reads and writes that do not flood the page cache (XFS 6.15+, ext4 6.17+, NFS 6.18+, block devices 7.3+). |
| `io_uring` | io_uring is the shared-ring asynchronous I/O interface: batch many reads and writes into one syscall, tag completions, and fall back when `kernel.io_uring_disabled` or seccomp blocks it. |
| `btrfs` | btrfs manages Btrfs subvolumes, snapshots, online checksum scrubs, and true CoW space usage. |
| `xfs_info` / `xfs_repair` | XFS tools for inspecting geometry, growing mounted XFS volumes (`xfs_growfs`), and repairing unmounted XFS disks. |
| `fstrim` | fstrim discards unused blocks on mounted SSDs, NVMe drives, and thin-provisioned VM volumes. |
| `mkswap` / `swapon` | Initialize, activate, prioritize, and deactivate swap partitions or swap files. |

## Suggested starting points

1. See devices: `lsblk`, `blkid`, `findmnt`.
2. Partition: `parted`/`fdisk`, then `mkfs`, `mount`/`umount`.
3. Check/repair & resize: `fsck` / `resize2fs` / `tune2fs` (ext*), `xfs_growfs` / `xfs_repair` (XFS), `btrfs` (Btrfs).
4. LVM trio: `pvs`, `vgs`, `lvs`.
5. Encryption & swap: `cryptsetup`, `mkswap`/`swapon`.
6. Space & SSD maintenance: `duf`/`dust`/`ncdu`, `fstrim`; health: `smartctl`.
7. Bulk I/O & async I/O: `RWF_DONTCACHE` and `io_uring`.

## Related parts

- Files and paths — `df`/`du` at the path layer
- Hardware — `nvme`, `hdparm`, and DMI
- Security — labels on mounted data

Continue with the individual command pages in this part.
