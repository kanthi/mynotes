# nvme

## Overview

`nvme` (from `nvme-cli`) is the standard command-line tool for inspecting, formatting, and managing **NVMe SSDs** directly over PCIe. Unlike `hdparm` (which targets ATA/SATA) or generic `smartctl`, `nvme` speaks native NVMe admin commands for namespace management, SMART/health logs, firmware updates, and secure erasures.

```bash
sudo apt install nvme-cli   # Debian/Ubuntu
sudo dnf install nvme-cli   # Fedora/RHEL
```

## Syntax

```bash
nvme <subcommand> [device] [options]
```

## Common Subcommands

| Subcommand | Description |
|------------|-------------|
| `list` | List all NVMe namespaces, models, serials, capacities, and firmware |
| `smart-log /dev/nvme0` | Show temperature, spare capacity, wear percentage, and media errors |
| `error-log /dev/nvme0` | Print controller error log entries |
| `id-ctrl /dev/nvme0` | Dump controller capabilities and features |
| `id-ns /dev/nvme0n1` | Show namespace LBA format, block size (512B vs 4K), and capacity |
| `format /dev/nvme0n1` | Low-level format / secure erase a namespace (**destructive!**) |
| `sanitize /dev/nvme0` | Cryptographic or block-level sanitize across the whole drive (**destructive!**) |

## Key Use Cases

1. Inventory NVMe drives, firmware revisions, and serial numbers before RMA or upgrades
2. Monitor drive wear (`percentage_used`), available spare, and temperature
3. Verify 4Kn vs 512e LBA sector sizing (`id-ns`) before partitioning
4. Cryptographically erase an NVMe SSD before decommissioning (`format -s 2` or `sanitize`)

## Safety

- **`nvme format` and `nvme sanitize` destroy all data immediately.** Always verify the exact device (`/dev/nvmeXnY`) with `nvme list` and `lsblk` first.
- Point controller commands (`smart-log`, `error-log`, `id-ctrl`) at `/dev/nvme0`, and namespace commands (`id-ns`, `format`) at `/dev/nvme0n1`.

## Examples with Explanations

### List all NVMe drives and firmware versions

```bash
sudo nvme list
```

Displays device node, SN, model, namespace usage, LBA format, and firmware revision in one table.

### Check SSD wear, temperature, and media errors

```bash
sudo nvme smart-log /dev/nvme0
```

Look closely at `critical_warning` (should be `0`), `temperature`, `available_spare`, `percentage_used`, and `media_errors`.

### Export health metrics as JSON

```bash
sudo nvme smart-log /dev/nvme0 -o json | jq '{temp: .temperature, used_pct: .percentage_used, media_errors: .media_errors}'
```

### Inspect supported LBA sector sizes

```bash
sudo nvme id-ns -H /dev/nvme0n1 | grep -E 'LBA Format|in use'
```

Shows whether the namespace is using 512-byte or 4096-byte sectors (`4Kn`).

## Notes & Pitfalls

- `nvme smart-log` reports temperature in **Kelvin** in raw JSON (`-o json`) on some versions, while human output shows Celsius—verify units before alerting.
- In VMs (AWS EBS Nitro, GCP), `/dev/nvme*` devices may be virtualized NVMe controllers where some vendor/wear fields are zeroed or synthetic.

## Related Commands

- `smartctl` — cross-bus SMART health checks (SATA, SAS, NVMe)
- `hdparm` — SATA/ATA parameters
- `lsblk` — block device tree and mount mapping
- `fstrim` — discard unused blocks on SSDs

## Additional Resources

- `man nvme`
- `man nvme-smart-log`
