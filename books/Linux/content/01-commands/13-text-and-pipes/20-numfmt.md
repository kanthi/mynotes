# numfmt

## Overview

`numfmt` (part of GNU coreutils) converts raw numbers to and from **human-readable units** (`K`, `M`, `G`, `T` or `Ki`, `Mi`, `Gi`, `Ti`) directly inside shell pipelines. It solves the classic problem where commands like `stat`, `ps`, or `ls` output raw bytes or kilobytes that are hard to scan—or where you need to parse `1.5G` back into bytes for arithmetic.

## Syntax

```bash
numfmt [options] [number...]
```

## Common Options

| Option | Description |
|--------|-------------|
| `--to=iec` | Convert bytes to binary SI-style units (`1024` -> `1.0K`, `Ki`/`Mi`/`Gi` with `iec-i`) |
| `--to=iec-i` | Convert bytes to explicit binary units (`1024` -> `1.0Ki`, `1048576` -> `1.0Mi`) |
| `--to=si` | Convert to decimal powers of 1000 (`1000` -> `1.0K`) |
| `--from=iec` / `--from=auto` | Parse human-readable input (`2.5Gi`) back into raw bytes |
| `--from-unit=1024` | Input numbers are already in KiB (e.g., `ps` RSS column) |
| `--field=N` | Convert only column `N` (default is field 1) |
| `--header[=N]` | Pass the first `N` header lines through untouched |
| `--padding=N` | Right-align (positive `N`) or left-align (negative `N`) output |

## Key Use Cases

1. Format raw byte columns from `ps`, `stat`, or `/proc` into readable MiB/GiB
2. Convert human-readable sizes (`4G`, `512M`) into exact bytes for scripts
3. Keep table headers intact (`--header`) when formatting piped command output

## Examples with Explanations

### Format `ps` RSS memory (KiB) into readable MiB/GiB

```bash
ps -eo pid,comm,rss --sort=-rss | head -n 11 | numfmt --header --to=iec-i --from-unit=1024 --field=3
```

`ps` reports `rss` in KiB (`--from-unit=1024`), and `--header` keeps the column header untouched.

### Convert human sizes back to raw bytes

```bash
numfmt --from=iec 2.5G 512M
# 2684354560
# 536870912
```

### Format raw byte counts with aligned columns

```bash
stat -c '%s %n' /var/log/*.log | sort -nr | numfmt --to=iec-i --padding=8
```

## Notes & Pitfalls

- Know the difference between `--to=si` (base 1000: `1K = 1000`) and `--to=iec` / `--to=iec-i` (base 1024: `1Ki = 1024`).
- `numfmt` is a GNU coreutils tool—on macOS (BSD) or minimal BusyBox/Alpine containers it may not be installed by default (`brew install coreutils` provides `gnumfmt`).

## Related Commands

- `sort` (`sort -h` sorts human-readable sizes directly)
- `column` — align tabular output
- `awk` — custom column calculations

## Additional Resources

- `man numfmt`
