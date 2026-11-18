# modprobe

## Overview

`modprobe` loads and unloads **Linux kernel modules** (`.ko` / `.ko.zst` drivers and filesystems), automatically resolving module dependencies from `modules.dep` and reading persistent options/blacklists from `/etc/modprobe.d/`. Always prefer `modprobe` over low-level `insmod`/`rmmod` for daily administration.

## Syntax

```bash
sudo modprobe [options] module [module_options...]
sudo modprobe -r [options] module...
```

## Common Options

| Option | Description |
|--------|-------------|
| `-r`, `--remove` | Unload a module and any unused dependencies it pulled in |
| `-v`, `--verbose` | Print each `insmod`/`rmmod` action performed |
| `-n`, `--dry-run` | Simulate loading/unloading without modifying the kernel |
| `--show-depends` | Print all dependency modules required by `module` |
| `-c`, `--showconfig` | Dump the effective `/etc/modprobe.d/` configuration and aliases |
| `-a`, `--all` | Load multiple modules listed on the command line |
| `-f`, `--force` | Ignore kernel version magic (**dangerous**, avoid in production) |

## Key Use Cases

1. Load a needed kernel subsystem on demand (`overlay`, `br_netfilter`, `wireguard`, `nbd`)
2. Inspect module dependencies (`--show-depends`) before troubleshooting
3. Unload a faulty or unused driver (`modprobe -r`) without rebooting
4. Blacklist problematic hardware drivers persistently via `/etc/modprobe.d/`

## Safety

- Unloading a module that is actively in use (`lsmod` `Used by > 0`) fails safely in `modprobe -r`; never force-unload (`rmmod -f`) on production hosts unless you accept a potential kernel panic.
- Out-of-tree modules (such as NVIDIA or ZFS DKMS) must be compiled for the exact running kernel (`uname -r`).
- When blacklisting a module that is loaded in the **initramfs** (e.g., GPU or storage drivers), rebuild initramfs (`update-initramfs -u` or `dracut -f`) afterward.

## Examples with Explanations

### Check loaded modules, then load or unload one

```bash
lsmod | grep overlay
sudo modprobe -v overlay
sudo modprobe -rv overlay
```

### Inspect dependencies and module metadata before loading

```bash
modprobe --show-depends xfs
modinfo xfs | head -n 20
```

`modinfo` shows the module's description, license, author, and accepted `parm:` options.

### Load a module with runtime parameters

```bash
sudo modprobe dummy numdummies=2
```

### Persist module loading at boot and blacklist unwanted modules

```bash
# Auto-load on boot:
echo 'br_netfilter' | sudo tee /etc/modules-load.d/k8s.conf

# Prevent a module from loading (use 'install ... /bin/false' to block even dependency loads):
echo 'blacklist pcspkr' | sudo tee /etc/modprobe.d/blacklist-pcspkr.conf
```

### List all available modules for the running kernel

```bash
find "/lib/modules/$(uname -r)" -name '*.ko*' | head -n 20
```

## Notes & Pitfalls

- `blacklist foo` in `/etc/modprobe.d/` only stops `udev` from auto-loading `foo` by hardware alias—another module that depends on `foo` can still pull it in unless you also add `install foo /bin/false`.
- After dropping a new `.ko` file manually into `/lib/modules/$(uname -r)/`, run `sudo depmod -a` so `modprobe` can find it.

## Related Commands

- `lsmod` / `modinfo` — list loaded modules and inspect parameters
- `depmod` — rebuild `modules.dep` dependency map
- `dmesg` — inspect kernel messages on module load/failure
- `uname -r` — verify the running kernel version

## Additional Resources

- `man modprobe`
- `man modprobe.d`
- `man modules-load.d`
