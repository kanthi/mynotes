# sensors

## Overview

`sensors` (from the `lm-sensors` package) reads hardware monitoring chips via `/sys/class/hwmon` to report **CPU/GPU temperatures, fan speeds (RPM), and motherboard voltages**. It is the first tool to reach for when diagnosing thermal throttling, noisy fans, or unexpected hardware shutdowns.

```bash
sudo apt install lm-sensors   # Debian/Ubuntu
sudo dnf install lm_sensors   # Fedora/RHEL
```

## Syntax

```bash
sensors [options] [chips...]
```

## Common Options

| Option | Description |
|--------|-------------|
| `-f` | Display temperatures in Fahrenheit instead of Celsius |
| `-j` | Output readings in JSON format (great for monitoring scripts) |
| `-u` | Raw output with individual sensor subfeature names |
| `-A` | Omit the adapter type line for cleaner output |
| `sensors-detect` | Interactive probe to load missing kernel hwmon modules |

## Key Use Cases

1. Check if a CPU is thermal throttling under load (`watch -n 1 sensors`)
2. Verify fan RPMs and pump speeds on bare-metal servers or workstations
3. Export hardware temperature telemetry in JSON (`sensors -j`) for Prometheus/scripts

## Examples with Explanations

### Watch temperatures and fan speeds live

```bash
watch -n 1 sensors
```

Refreshes every second—run alongside a stress test or build to see whether temperatures hit `high` or `crit` thresholds.

### Probe and load missing sensor kernel modules

```bash
sudo sensors-detect --auto
```

Detects I2C/SMBus and Super-I/O chips and loads the corresponding kernel drivers (like `coretemp`, `k10temp`, `nct6775`).

### Machine-readable JSON output for scripts

```bash
sensors -j | jq '."coretemp-isa-0000"."Package id 0".temp1_input'
```

Reads the CPU package temperature cleanly without brittle `grep`/`awk` parsing.

## Notes & Pitfalls

- Virtual machines and cloud instances usually expose no physical `hwmon` chips—`sensors` will report `No sensors found!`.
- NVMe SSD temperatures are often exposed under `nvme-pci-*` in `sensors` (or via `nvme smart-log`).
- On laptops, fan control or extra sensors may require vendor modules (`thinkpad_acpi`, `dell_smm_hwmon`).

## Related Commands

- `nvme` — NVMe drive temperature and SMART health
- `smartctl` — SATA/NVMe disk SMART attributes
- `dmidecode` — chassis and board info

## Additional Resources

- `man sensors`
- `man sensors-detect`
