# trap

## Overview

`trap` registers a command or shell function to run automatically when the shell receives a **signal** (`INT`, `TERM`, `HUP`) or hits a pseudo-signal (`EXIT`, `ERR`, `DEBUG`). It is the standard way to guarantee temporary files, lock files, or background child processes are cleaned up whether a script succeeds, fails, or is interrupted with `Ctrl+C`.

## Syntax

```bash
trap 'command' SIGNAL [SIGNAL...]
trap - SIGNAL     # reset to default behavior
trap '' SIGNAL    # ignore signal
trap -p           # print currently registered traps
```

## Common Signals & Pseudo-Signals

| Signal | Trigger |
|--------|---------|
| `EXIT` | Always runs when the shell exits (normal end, `set -e` failure, or signal) |
| `ERR` | Runs whenever a command returns non-zero (pair with `set -eE`) |
| `INT` | Interrupt (`Ctrl+C` / `SIGINT`) |
| `TERM` | Termination request (`kill` default / `SIGTERM`) |
| `HUP` | Hangup (`SIGHUP`, often used to reload daemon configs) |

## Key Use Cases

1. Always remove temporary directories (`mktemp -d`) when a script finishes or crashes
2. Log the exact line number and failing command on `ERR`
3. Gracefully stop background child processes when the parent script exits

## Examples with Explanations

### Guaranteed temp directory cleanup on `EXIT`

```bash
#!/usr/bin/env bash
set -euo pipefail

workdir=$(mktemp -d)
cleanup() {
  rm -rf "$workdir"
}
trap cleanup EXIT

cp /etc/hosts "$workdir/hosts.bak"
```

Using `EXIT` handles normal completion, `set -e` aborts, and `SIGINT`/`SIGTERM` cleanly in one place without duplicating cleanup logic.

### Log failing line number with `ERR`

```bash
set -euo pipefail
trap 'echo "Error on line $LINENO (exit $?)" >&2' ERR
```

### Ignore a signal temporarily or reset a trap

```bash
trap '' INT          # ignore Ctrl+C during critical section
# ... critical work ...
trap - INT           # restore default Ctrl+C behavior
```

## Notes & Pitfalls

- Use **single quotes** (`trap "rm -rf '$dir'" EXIT` vs `trap 'cleanup' EXIT`) intentionally: single quotes defer variable expansion until the trap fires; double quotes expand immediately when `trap` is defined. A helper function (`trap cleanup EXIT`) avoids quoting headaches altogether.
- `SIGKILL` (`kill -9`) and `SIGSTOP` **cannot** be caught or ignored by `trap`.
- In subshells (`( ... )`), traps are reset unless inherited; use `set -E` (`errtrace`) so `ERR` traps propagate into functions and subshells.

## Related Commands

- `set` — strict mode (`set -euo pipefail`)
- `kill` — send signals to processes
- `mktemp` — create safe temporary files/directories

## Additional Resources

- `help trap` (in Bash)
- `man 7 signal`
