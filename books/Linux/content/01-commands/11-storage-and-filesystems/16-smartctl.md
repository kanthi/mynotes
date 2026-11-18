# smartctl

## Overview

`smartctl` (from **smartmontools**) queries disk **S.M.A.R.T.** (Self-Monitoring, Analysis, and Reporting Technology) health data: overall pass/fail assessment, wear/temperature attributes, error logs, and hardware self-tests. It works across SATA/SAS HDDs, SATA SSDs, and NVMe drives.

```bash
sudo apt install smartmontools   # Debian/Ubuntu
sudo dnf install smartmontools   # Fedora/RHEL
```

## Syntax

```bash
sudo smartctl [options] device
```

## Common Options

| Option | Description |
|--------|-------------|
| `-H`, `--health` | Overall SMART health status (`PASSED` / `FAILED`) |
| `-i`, `--info` | Drive model, serial number, firmware, and capacity |
| `-A`, `--attributes` | Vendor-specific SMART attributes table (temperatures, reallocated sectors) |
| `-a`, `--all` | Standard health, info, attributes, and error/self-test logs |
| `-x`, `--xall` | Extended report (includes SATA PHY errors and NVMe logs) |
| `-l error` / `-l selftest` | Print device error log or self-test history |
| `-t short\|long` | Launch an offline/captive background self-test |
| `-d TYPE` | Specify controller passthrough type (`nvme`, `sat`, `megaraid,0`, `cciss,0`) |
| `--scan` | Detect all SMART-capable devices on the system |

## Key Use Cases

1. Triage suspected failing disks before data loss (`-H`, `-A`, `-l error`)
2. Run scheduled or on-demand short/long surface tests (`-t short` / `-t long`)
3. Audit drive serial numbers, firmware versions, and power-on hours (`-i`, `-a`)
4. Query individual physical drives behind hardware RAID controllers (`-d megaraid,N`)

## Safety

- Long self-tests (`-t long`) read every sector on the disk and can take hours on multi-terabyte HDDs, adding I/O latency—schedule them during maintenance windows.
- On hardware RAID controllers, querying `/dev/sda` directly only sees the logical volume; use `-d megaraid,N` or vendor tools to inspect the physical drives underneath.
- A `PASSED` status from `-H` does **not** guarantee a healthy drive—always check reallocated/pending sectors and media errors.

## Examples with Explanations

### Discover devices and check overall health

```bash
sudo smartctl --scan
sudo smartctl -H -i /dev/sda
```

### Inspect full SMART report or extended diagnostics

```bash
sudo smartctl -a /dev/sda | less
sudo smartctl -x /dev/nvme0 | less
```

### Watch key failure-predictor attributes

```bash
sudo smartctl -A /dev/sda | grep -iE 'reallocated|pending|uncorrect|wear|temperature|power_on'
```

Non-zero `Reallocated_Sector_Ct`, `Current_Pending_Sector`, or `Offline_Uncorrectable` values are early warnings of a dying HDD.

### Run a short self-test and check the result

```bash
sudo smartctl -t short /dev/sda
# Wait ~2 minutes, then inspect:
sudo smartctl -l selftest /dev/sda
```

### Query a physical drive behind a MegaRAID controller

```bash
sudo smartctl -a -d megaraid,0 /dev/sda
```

## Understanding Output

Key columns in `smartctl -A` for SATA/HDDs:

| Column | Meaning |
|--------|---------|
| `ID#` / `ATTRIBUTE_NAME` | Metric identifier (e.g., `5 Reallocated_Sector_Ct`, `194 Temperature_Celsius`) |
| `VALUE` / `WORST` / `THRESH` | Normalized score (starts at 100/200 and drops toward `THRESH` as it degrades) |
| `RAW_VALUE` | Actual physical count (sectors, hours, Celsius) — usually what you want to read |

## Notes & Pitfalls

- USB-to-SATA enclosures often block SMART commands unless you pass `-d sat` (`sudo smartctl -a -d sat /dev/sdb`).
- Cloud virtual disks (EBS, GCE PD) do not expose physical SMART attributes.

## Related Commands

- `nvme` (`nvme smart-log`) — native NVMe health and namespace diagnostics
- `dmesg` — check kernel I/O and medium error messages
- `hdparm` — SATA identify and power settings
- `smartd` — background daemon that emails alerts when SMART attributes degrade

## Additional Resources

- `man smartctl`
- `man smartd.conf`
