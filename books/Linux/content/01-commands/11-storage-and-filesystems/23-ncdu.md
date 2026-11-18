# ncdu

## Overview

`ncdu` (NCurses Disk Usage) is an interactive terminal disk-usage analyzer. Instead of running `du -sh *` repeatedly down one directory at a time, `ncdu` scans a tree once and lets you drill up and down folders sorted by size—making it one of the fastest tools for tracking down what filled a root filesystem.

```bash
sudo apt install ncdu   # Debian/Ubuntu
sudo dnf install ncdu   # Fedora/RHEL
```

## Syntax

```bash
ncdu [options] [path]
```

## Common Options & Interactive Keys

| Option / Key | Description |
|--------------|-------------|
| `-x` | Stay on one filesystem (do not cross into `/proc`, `/sys`, or network mounts) |
| `-r` | Read-only mode (disables the `d` delete key) |
| `-q` | Quiet mode (updates progress every 2 seconds instead of 100 ms—faster over slow SSH) |
| `-o FILE` | Export scan results to a file (great for scanning a busy prod host and viewing locally) |
| `-f FILE` | Load and browse a previously exported scan file |
| `--exclude PATTERN` | Exclude matching directories or files |
| `Arrow keys` / `Enter` | Navigate directories |
| `d` | Delete selected file or directory (unless `-r` is set) |
| `s` / `n` / `C` | Sort by size / name / item count |
| `a` | Toggle apparent size vs. actual disk usage (useful for sparse files) |

## Key Use Cases

1. Quickly find which directory, log, or cache filled `/` on a full server
2. Scan a remote production server (`ncdu -x -o`) and inspect the report offline on your laptop
3. Safely browse disk usage in read-only mode (`-r`) without risking accidental `d` deletions

## Safety

- Always pass `-x` when scanning `/` so `ncdu` doesn't walk `/proc`, `/sys`, `/dev`, or mounted NFS/SMB shares.
- Use `-r` (read-only) when investigating production servers if you only want to inspect, so an accidental `d` keypress cannot delete files.

## Examples with Explanations

### Scan the root filesystem safely (single filesystem, read-only)

```bash
sudo ncdu -rx /
```

### Scan your home directory and exclude caches or `.git`

```bash
ncdu -x --exclude .git --exclude node_modules "$HOME"
```

### Export a scan on a remote server and view it locally

```bash
# On the remote server:
sudo ncdu -1xo- / | gzip > /tmp/root-scan.ncdu.gz

# Copy to your laptop and browse interactively:
gzip -dc root-scan.ncdu.gz | ncdu -f -
```

Zero impact on the remote terminal while you drill into directories locally.

### Quick quiet scan over slow SSH

```bash
sudo ncdu -qx /var
```

Reduces terminal redraw traffic over high-latency SSH links.

## Notes & Pitfalls

- Run with `sudo` when scanning `/var` or `/`; without root, `ncdu` skips unreadable directories (showing `[permission denied]` errors) and under-reports their size.
- `ncdu` counts hardlinked files only once (like `du`), and `a` toggles apparent size vs. allocated blocks (helpful when sparse VM images look huge in `ls -l`).
- `ncdu` cannot see **deleted-but-still-open** files holding disk space; if `df` says 100% full but `ncdu -x /` only accounts for half the disk, check `sudo lsof +L1`.

## Related Commands

- `du` / `dust` — non-interactive directory size trees
- `df` / `duf` — filesystem-level free space
- `lsof` (`lsof +L1`) — find deleted files still held open by processes

## Additional Resources

- `man ncdu`
