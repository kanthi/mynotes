# lsusb

## Overview

`lsusb` lists **USB buses and connected USB devices** (vendor/product IDs, speeds, device classes, and descriptors). It is the first diagnostic step when troubleshooting USB-C docks, storage drives, hardware security keys, or USB-to-serial adapters (`ttyUSB0`, `ttyACM0`) before diving into drivers or `udev` rules.

```bash
sudo apt install usbutils   # Debian/Ubuntu
sudo dnf install usbutils   # Fedora/RHEL
```

## Syntax

```bash
lsusb [options]
```

## Common Options

| Option | Description |
|--------|-------------|
| `-t`, `--tree` | Show the physical USB hub/port hierarchy, speeds (`480M`, `5000M`, `10000M`), and bound kernel drivers |
| `-v`, `--verbose` | Print full USB descriptors (endpoints, power draw, interfaces) |
| `-d VENDOR:PRODUCT` | Filter output to a specific hex `ID` (e.g., `-d 0781:5567`) |
| `-s [[BUS]:][DEV]` | Filter by bus and/or device number (e.g., `-s 001:005`) |
| `-D DEVICE` | Query a specific `/dev/bus/usb/BBB/DDD` node directly |

## Key Use Cases

1. Confirm whether the kernel physically enumerated a USB device (`lsusb`)
2. Check link speed (`480M` USB 2.0 vs `5000M` USB 3.0 vs `10000M` USB 3.2) and bound driver (`lsusb -t`)
3. Grab `vendor:product` hex IDs for writing custom `udev` rules or VM passthrough filters
4. Inspect USB power draw (`MaxPower`) when debugging flaky docks or bus-powered drives

## Examples with Explanations

### List all USB devices and their physical hub tree

```bash
lsusb
lsusb -t
```

`-t` reveals nested USB-C hubs, negotiated speeds (`5000M`), and which kernel driver (`usb-storage`, `cdc_acm`, `usbhid`) claimed each interface.

### Filter by Vendor:Product ID or bus/device

```bash
lsusb -d 1d6b:0003
lsusb -s 002:004
```

### Inspect power draw and full descriptors

```bash
sudo lsusb -v -d 0781:5567 | grep -E 'idVendor|idProduct|MaxPower|iProduct'
```

### Correlate hot-plug events with kernel driver logs

```bash
sudo dmesg -w
# Plug in the USB device and watch ttyUSB/ttyACM or nvme/sd assignment
```

### Locate character/block device nodes created by USB devices

```bash
ls -l /dev/ttyUSB* /dev/ttyACM* /dev/bus/usb/*/* 2>/dev/null
```

## Understanding Output

Typical `lsusb` line:

```text
Bus 002 Device 004: ID 0781:5567 SanDisk Corp. Cruzer Blade
```

| Field | Meaning |
|-------|---------|
| `Bus 002 Device 004` | USB host controller bus and assigned device number |
| `0781:5567` | Hex `idVendor:idProduct` from the USB-IF registry |
| `SanDisk Corp. ...` | Human-readable vendor and product name from `usb.ids` |

## Notes & Pitfalls

- `lsusb -v` requires `sudo` to read all device descriptors; without root, some fields are omitted.
- **USBGuard** or kernel authorization policies may block a device from binding a driver even though it appears in `lsusb`.
- In VMs, `lsusb` only sees virtualized or explicitly passed-through devices—check the **host** if a device doesn't appear inside the guest.
- Frequent disconnects or dropping from `5000M` to `480M` usually indicate a damaged cable or underpowered hub.

## Related Commands

- `lspci` — inspect PCI/PCIe controllers and host buses
- `lshw` — full system hardware inventory
- `dmesg` / `journalctl -k` — kernel USB enumeration and driver errors
- `udevadm` (`udevadm info`) — inspect `/dev` udev properties and rules
- `modprobe` — load/inspect kernel USB drivers

## Additional Resources

- `man lsusb`
