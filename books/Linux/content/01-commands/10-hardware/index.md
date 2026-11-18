---
title: Intro
---

# Intro

Inventory and inspect physical devices: whole-system reports, PCI devices, DMI/SMBIOS data, thermal sensors, NVMe SSDs, and the hardware RTC. Use before opening tickets, diagnosing throttling, or swapping hardware.

## Commands in this part

| Command | Role |
|---------|------|
| `lshw` | lshw (list hardware) probes the system and prints a hardware tree: CPU, memory, storage, network, firmware, and buses. |
| `lspci` | lspci lists PCI and PCI Express devices: NICs, GPUs, storage controllers, USB controllers, bridges, and more. |
| `dmidecode` | dmidecode dumps DMI/SMBIOS tables: manufacturer, product name, serial numbers, BIOS version, memory slot population,… |
| `hdparm` | hdparm gets and sets ATA/SATA disk parameters: identify data, power management, read-ahead, write-cache, and… |
| `sensors` | sensors reads motherboard/CPU/GPU thermal sensors, fan RPMs, and voltages via `lm-sensors`. |
| `nvme` | nvme inspects NVMe SSD namespaces, firmware, wear (`smart-log`), and performs secure erasures. |
| `hwclock` | hwclock reads and sets the motherboard's battery-backed Real-Time Clock (RTC). |

## Quick Triage Recipes

```bash
# 1. Live thermal & fan check while debugging CPU throttling
watch -n 1 sensors

# 2. Check NVMe SSD wear and media errors
sudo nvme list && sudo nvme smart-log /dev/nvme0

# 3. Grab board serial number and BIOS version for a support ticket
sudo dmidecode -t system -t bios
```

## Suggested starting points

1. Broad inventory: `lshw`.
2. PCI devices: `lspci`.
3. Firmware/DMI: `dmidecode`.
4. Thermals and fans: `sensors`.
5. Disks: `nvme` (NVMe SSDs) or `hdparm` (SATA); SMART health is under Storage (`smartctl`).
6. Hardware RTC: `hwclock`.

## Related parts

- System information — quick CPU/memory summaries (`lscpu`, `lsmem`, `lsusb`)
- Storage and filesystems — block devices, `smartctl`, `fstrim`
- Services and runtime — kernel modules via `modprobe`/`lsmod`

Continue with the individual command pages in this part.
