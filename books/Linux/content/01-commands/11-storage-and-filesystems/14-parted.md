# parted

## Overview

`parted` creates, resizes, and inspects **partition tables** (GPT and MBR/msdos) on block devices. Unlike `fdisk`, `parted` supports disks larger than 2 TiB out of the box with GPT, supports non-interactive scripting via `-s` (`--script`), and can resize partition boundaries cleanly with `resizepart`. Mistakes overwrite partition metadata immediately—always double-check the target device before running write commands.

## Syntax

```bash
sudo parted [options] device [command...]
sudo parted -l
```

## Common Options & Subcommands

| Command / Option | Description |
|------------------|-------------|
| `-s`, `--script` | Non-interactive mode (never prompts for confirmation) |
| `-a optimal` | Align new partitions to optimal I/O boundaries (1 MiB) |
| `-l` | List partition tables on all detected block devices |
| `print` | Show the partition table, disk size, and partition flags |
| `mklabel gpt\|msdos` | Create a new partition table (**destroys existing table!**) |
| `mkpart NAME [FS] START END` | Create a partition without formatting a filesystem |
| `resizepart N END` | Move the end boundary of partition `N` |
| `rm N` | Delete partition number `N` |
| `set N FLAG on\|off` | Toggle flags such as `esp`, `boot`, `lvm`, or `raid` |
| `align-check optimal N` | Verify partition `N` is properly aligned for SSDs/RAID |

## Key Use Cases

1. Initialize a new disk with a GPT partition table (`mklabel gpt`)
2. Provision partitions reproducibly in Ansible/cloud-init scripts (`parted -s`)
3. Expand a partition boundary before running `resize2fs` or `xfs_growfs`
4. Verify 1 MiB / optimal alignment on NVMe and SSDs (`align-check optimal`)

## Safety

- **Double-check** `lsblk -o NAME,SIZE,MODEL,SERIAL` or `/dev/disk/by-id/...` before running `mklabel` or `rm`.
- `mklabel` immediately overwrites the partition table header on disk—unlike `fdisk`, `parted` applies changes **immediately** without a final `w` (write) confirmation!
- Always unmount partitions and deactivate swap/LVM on a disk before deleting or shrinking partitions.

## Examples with Explanations

### Inspect all disks and a specific partition table

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,UUID
sudo parted -l
sudo parted /dev/sdb print
```

### Script a GPT disk with a single aligned partition

```bash
sudo parted -s -a optimal /dev/sdb mklabel gpt
sudo parted -s -a optimal /dev/sdb mkpart primary ext4 1MiB 100%
sudo parted /dev/sdb align-check optimal 1
sudo mkfs.ext4 -L data /dev/sdb1
```

`parted` only creates the partition entry (`mkpart`); `mkfs.ext4` writes the actual filesystem.

### Create an EFI System Partition (ESP) + root partition

```bash
sudo parted -s /dev/nvme0n1 mklabel gpt
sudo parted -s /dev/nvme0n1 mkpart ESP fat32 1MiB 513MiB
sudo parted -s /dev/nvme0n1 set 1 esp on
sudo parted -s /dev/nvme0n1 mkpart root ext4 513MiB 100%
```

Sets the `esp` flag so UEFI firmware recognizes `/dev/nvme0n1p1` as the boot partition.

### Expand a partition to fill a newly grown cloud disk

```bash
sudo parted -s /dev/sda resizepart 1 100%
sudo resize2fs /dev/sda1
```

Grows partition 1 to 100% of the disk, then expands the ext4 filesystem inside it.

### Interactive session with explicit units

```bash
sudo parted /dev/sdb
# (parted) unit GiB
# (parted) print
# (parted) quit
```

## Notes & Pitfalls

- Changes in `parted` are **written immediately** as you press Enter—there is no undo or `q` without saving.
- The kernel may not see new partitions on a busy disk until you run `sudo partprobe /dev/sdb` (or `partx -u /dev/sdb`).
- NVMe partitions include a `p` before the partition number (`/dev/nvme0n1p1`), whereas SATA/SCSI uses `/dev/sdb1`.

## Related Commands

- `fdisk` / `gdisk` — interactive MBR/GPT partitioners (wait for `w` before writing)
- `lsblk` / `blkid` — inspect block devices, labels, and UUIDs
- `mkfs` / `mount` — format and mount created partitions
- `wipefs` — clear old filesystem/RAID signatures before repartitioning

## Additional Resources

- `man parted`
- `info parted`
