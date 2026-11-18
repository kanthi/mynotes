# hwclock

## Overview

`hwclock` reads and sets the motherboard's **hardware Real-Time Clock (RTC)** (`/dev/rtc0`), which keeps time using a CMOS battery even when the machine is powered off. At boot, Linux reads the RTC to initialize the system clock before NTP (`systemd-timesyncd` or `chronyd`) syncs over the network.

## Syntax

```bash
hwclock [function] [options]
```

## Common Options

| Option | Description |
|--------|-------------|
| `-r`, `--show` | Read and print the hardware clock (default action) |
| `-s`, `--hctosys` | Set the system clock from the hardware clock |
| `-w`, `--systohc` | Write the current system clock into the hardware clock |
| `-u`, `--utc` | Interpret the hardware clock as UTC (Linux standard) |
| `--localtime` | Interpret the hardware clock as local time (legacy Windows dual-boot) |
| `--verbose` | Show clock source, drift, and UTC/localtime mode |

## Key Use Cases

1. Check if the hardware RTC has drifted or lost time (sign of a dying CMOS battery)
2. Sync an NTP-corrected system clock back to the RTC (`hwclock -w`)
3. Diagnose dual-boot time jumps between Linux (UTC) and Windows (localtime)

## Safety

- On systems running `chronyd` or `systemd-timesyncd`, the kernel usually syncs the RTC automatically every 11 minutes—use `hwclock -w` manually only when needed (e.g., offline servers or after a large manual `date` adjustment).
- Always keep the RTC in **UTC** on Linux servers (`timedatectl set-local-rtc 0`).

## Examples with Explanations

### Read the hardware clock and compare with system time

```bash
sudo hwclock --show --verbose
date -u
```

Shows the RTC time alongside whether `/etc/adjtime` treats it as UTC or localtime.

### Write system time to the hardware clock

```bash
sudo hwclock --systohc --utc
```

Saves the current kernel time to the RTC in UTC so the next reboot starts accurate before NTP connects.

### Set system time from the hardware clock (e.g., in rescue mode)

```bash
sudo hwclock --hctosys
```

## Notes & Pitfalls

- If `hwclock` resets to `1970-01-01` or `2000-01-01` after power loss, replace the motherboard CMOS battery (CR2032).
- Containers and some cloud VMs do not expose `/dev/rtc0` or block RTC writes (`Operation not permitted`).
- Prefer `timedatectl` for day-to-day timezone and NTP status on systemd hosts.

## Related Commands

- `timedatectl` — systemd time, timezone, and NTP control
- `date` — print or set system software clock

## Additional Resources

- `man hwclock`
