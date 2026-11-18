# fstrim

## Overview

`fstrim` sends `FITRIM` (discard) ioctls to a mounted filesystem so the underlying **SSD, NVMe drive, or thin-provisioned VM disk** knows which freed blocks are no longer in use. This keeps SSD write performance high, reduces wear, and lets thin-provisioned SANs/VMs reclaim freed host space.

## Syntax

```bash
fstrim [options] <mountpoint>
fstrim -a [options]
```

## Common Options

| Option | Description |
|--------|-------------|
| `-v`, `--verbose` | Print the number of bytes trimmed per mountpoint |
| `-a`, `--all` | Trim all mounted filesystems that support discard |
| `-A`, `--fstab` | Trim filesystems listed in `/etc/fstab` (used by `fstrim.service`) |
| `-n`, `--dry-run` | Print what would be trimmed without issuing discard commands |
| `-m`, `--minimum bytes` | Skip contiguous free extents smaller than `bytes` |

## Key Use Cases

1. Run a weekly trim via `fstrim.timer` (preferred over continuous `discard` mount option)
2. Reclaim thin-provisioned VM disk space (`qcow2`, LVM thin pools, cloud volumes) after deleting large files
3. Verify whether a disk/controller actually supports TRIM/UNMAP

## Safety

- On **LUKS-encrypted** volumes, TRIM is disabled by default for security (it can leak which blocks are empty); only enable `allow-discards` in `/etc/crypttab` if you accept that trade-off.
- Do not run `fstrim` continuously in tight loops—a weekly `fstrim.timer` is the Linux standard.

## Examples with Explanations

### Trim all supported mounted filesystems verbosely

```bash
sudo fstrim -av
```

Outputs e.g. `/home: 14.2 GiB (15247130624 bytes) trimmed` for each supported mount.

### Trim a single mountpoint

```bash
sudo fstrim -v /var/lib/docker
```

### Check if the weekly systemd timer is enabled

```bash
systemctl status fstrim.timer
```

Most modern distros ship `fstrim.timer` (weekly) so you do not need `discard` in `/etc/fstab`.

## Notes & Pitfalls

- If `fstrim` prints `the discard operation is not supported`, check `lsblk --discard` (`DISC-MAX` will be `0B` if the controller, USB bridge, or VM hypervisor does not pass UNMAP through).
- Periodic `fstrim` is generally preferred over the `discard` mount option, which can add latency to every file deletion.

## Related Commands

- `lsblk` (`lsblk --discard`) — check hardware discard support
- `nvme` — NVMe drive management
- `mount` — filesystem mount options

## Additional Resources

- `man fstrim`
