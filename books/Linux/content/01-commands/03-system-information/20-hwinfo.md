# hwinfo

## Overview

`hwinfo` probes system hardware using `libhd` and prints detailed information across CPUs, RAM, disks, PCI/USB buses, network cards, and BIOS/UEFI. While `lshw`, `lspci`, and `dmidecode` are more commonly preinstalled on minimal servers, `hwinfo --short` provides one of the cleanest, scannable hardware summaries available on openSUSE, Ubuntu, and Debian workstations.

```bash
sudo apt install hwinfo   # Debian/Ubuntu
sudo zypper install hwinfo # openSUSE (often preinstalled)
```

## Syntax

```bash
sudo hwinfo [options]
```

## Common Options

| Option | Description |
|--------|-------------|
| `--short` | Print a concise, categorized one-line-per-device summary |
| `--cpu` / `--memory` | Inspect CPU cores/flags or physical memory |
| `--disk` / `--block` | List disks, controllers, and block devices |
| `--network` / `--netcard` | Show network interfaces, MAC addresses, and drivers |
| `--gfxcard` | Show GPU model, VRAM, and active driver |
| `--pci` / `--usb` | Probe PCI or USB buses |
| `--all` | Full exhaustive hardware probe (slow, verbose) |
| `--log FILE` | Write probe debug log to `FILE` |

## Key Use Cases

1. Generate a clean one-screen hardware inventory (`sudo hwinfo --short`)
2. Identify GPU, NIC, or audio chipsets and which kernel driver is bound (`--gfxcard`, `--netcard`)
3. Capture a complete hardware diagnostic bundle (`--all --log`) for support tickets

## Safety

- Running `hwinfo --all` performs deep hardware probing across buses and can take several seconds (or occasionally trigger quirks on unusual legacy hardware)—prefer targeted flags like `--short`, `--disk`, or `--netcard` on production servers.

## Examples with Explanations

### Clean one-screen hardware summary

```bash
sudo hwinfo --short
```

Groups CPU, keyboard, mouse, graphics, storage, and network adapters into a readable overview.

### Inspect network adapters and active kernel drivers

```bash
sudo hwinfo --netcard
```

Shows PCI ID, MAC address, link status, and the exact driver activation command (`modprobe ...`).

### Check disks and GPU details

```bash
sudo hwinfo --disk --short
sudo hwinfo --gfxcard
```

### Save a full hardware report to a file

```bash
sudo hwinfo --all --log /tmp/hwinfo.log > /tmp/hardware-report.txt
```

## Notes & Pitfalls

- `hwinfo` needs `sudo` for full BIOS, DMI, and disk serial probing; unprivileged runs omit sensitive or privileged fields.
- On minimal RHEL/Fedora servers, `hwinfo` is often not packaged—use `lshw`, `lspci`, `lsblk`, and `dmidecode` there instead.

## Related Commands

- `lshw` — cross-distro hardware tree
- `lspci` / `lsusb` — fast PCI and USB bus listings
- `dmidecode` — SMBIOS/DMI firmware and RAM slot tables
- `lscpu` / `lsblk` — lightweight CPU and disk summaries

## Additional Resources

- `man hwinfo`
- `https://github.com/openSUSE/hwinfo`
