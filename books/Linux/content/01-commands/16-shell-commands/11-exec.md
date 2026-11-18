# exec

## Overview

`exec` is a shell builtin with two distinct superpowers:
1. **Replace the current shell process** with another command (no `fork`, same PID—crucial in container `ENTRYPOINT` scripts so signals like `SIGTERM` reach your app directly).
2. **Redirect file descriptors** (`stdout`, `stderr`, or custom FDs) for the remainder of the current script without spawning a subshell.

## Syntax

```bash
exec [-cl] [-a name] [command [args...]]
exec [redirection...]
```

## Common Options

| Option | Description |
|--------|-------------|
| `exec cmd args` | Replace the shell process with `cmd` (never returns on success) |
| `exec > file 2>&1` | Redirect all subsequent `stdout` and `stderr` in the script to `file` |
| `exec 3> file` | Open a custom file descriptor (`3`) for writing |
| `exec 3>&-` | Close file descriptor `3` |
| `-a name` | Pass `name` as `argv[0]` to the executed command |
| `-c` | Execute `command` with an empty environment |

## Key Use Cases

1. Container `ENTRYPOINT` wrappers so PID 1 is the application, not `/bin/sh`
2. Log an entire script's output to a file (and/or `tee`) with one line at the top
3. Manage lock file descriptors (`exec 9>/var/lock/myjob.lock; flock -n 9`)

## Examples with Explanations

### Container entrypoint pattern (receive `SIGTERM` directly)

```bash
#!/usr/bin/env bash
set -euo pipefail

# Run pre-flight setup, then hand PID 1 over to the server
migrate_db
exec /usr/local/bin/my-server --port 8080
```

Without `exec`, Bash stays PID 1 and may not forward `SIGTERM` from `docker stop` or Kubernetes to `my-server`.

### Redirect all script output to a log file and terminal

```bash
exec > >(tee -a /var/log/deploy.log) 2>&1
echo "Starting deploy at $(date)"
```

Every `echo` and command error below this line automatically goes to both stdout and `/var/log/deploy.log`.

### Open and close a lock file descriptor with `flock`

```bash
exec 9>/var/lock/backup.lock
flock -n 9 || { echo "Already running" >&2; exit 1; }
# ... critical work ...
exec 9>&-
```

## Notes & Pitfalls

- Because `exec cmd` replaces the shell process, **any lines after `exec cmd` will never run** unless `exec` itself fails to find/run the binary.
- When redirecting `exec > >(tee ...)` in Bash, `tee` runs asynchronously—the script may exit slightly before `tee` finishes flushing the last line unless you wait or sync.

## Related Commands

- `flock` — file locking using `exec` file descriptors
- `tee` — duplicate output streams
- `source` — run a script in the current shell without replacing it

## Additional Resources

- `help exec` (in Bash)
- `man 3 exec`
