# tac & rev

## Overview

`tac` and `rev` (from GNU coreutils and `util-linux`) both reverse text, but along different axes:
- **`tac`** (`cat` spelled backward) reverses **lines** (prints the last line first, up to the first line).
- **`rev`** reverses **characters** within each line (turns `abc` into `cba`).

`tac` is especially handy for scanning huge log files **newest-first** without loading the whole file into an editor, and `rev` is a classic pipeline trick for grabbing the *last* field when the number of fields varies per line.

## Syntax

```bash
tac [options] [file...]
rev [file...]
```

## Common Options

| Tool / Option | Description |
|---------------|-------------|
| `tac -s SEP` | Use a custom record separator instead of newline (e.g., multi-line records) |
| `tac -b` | Attach separator **before** each record rather than after |
| `tac -r` | Treat the `-s` separator as a regular expression |
| `rev` | Reverse characters on every line from file(s) or stdin |

## Key Use Cases

1. Read a log file from newest entry to oldest (`tac app.log | head -n 50`)
2. Find the **most recent** match in a huge log and stop immediately (`tac huge.log | grep -m 1 'ERROR'`)
3. Extract the last path component or last space-delimited field with `rev | cut -d/ -f1 | rev`
4. Reverse multi-line blocks (such as paragraphs or stack traces) using `tac -s`

## Examples with Explanations

### Find the single most recent error in a huge log instantly

```bash
tac /var/log/app.log | grep -m 1 'OutOfMemory'
```

`tac` reads the file backward from the end, and `grep -m 1` exits immediately on the first hit—much faster than scanning gigabytes from the top.

### View the last 30 lines of a log in reverse chronological order

```bash
tac /var/log/syslog | head -n 30
```

### Extract the last column when lines have varying numbers of fields (`rev | cut | rev`)

```bash
echo -e "a/b/c.txt\nx/y/z/final.log" | rev | cut -d/ -f1 | rev
# c.txt
# final.log
```

Useful when `cut -f` alone cannot count from the right (whereas `awk -F/ '{print $NF}'` also works).

### Reverse multi-line records separated by blank lines

```bash
tac -s $'\n\n' records.txt
```

### Check for palindromes or reverse strings in a pipeline

```bash
echo "linux" | rev
# xunil
```

## Notes & Pitfalls

- `tac` can seek backward efficiently on **regular files**, but when reading from a **pipe** (`cmd | tac`), it must buffer the entire stream in memory/temp space before printing line 1—avoid piping multi-gigabyte streams into `tac`.
- `rev` is UTF-8 aware on modern `util-linux` (reversing multi-byte characters cleanly rather than corrupting bytes).

## Related Commands

- `cat` — print files forward
- `tail` — read the last N lines in normal forward order
- `awk` (`$NF`) — access the last field directly without `rev | cut | rev`

## Additional Resources

- `man tac`
- `man rev`
