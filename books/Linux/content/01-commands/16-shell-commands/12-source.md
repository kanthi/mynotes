# source (.)

## Overview

`source` (and its POSIX equivalent `.`) reads and executes commands from a file **inside the current shell environment** rather than spawning a child subshell. This is how you load environment variables (`.env`), reload `~/.bashrc`, or import reusable Bash function libraries into scripts.

## Syntax

```bash
source filename [arguments...]
. filename [arguments...]
```

## Key Use Cases

1. Reload shell configuration after editing `~/.bashrc` or `~/.zshrc`
2. Load environment variables from a `.env` or config file into the current shell
3. Import shared helper functions into multiple Bash scripts

## Safety

- **Never `source` untrusted files**—unlike executing a script in a subshell, a sourced file runs with full access to modify your current shell's variables, `PATH`, aliases, and working directory.
- If a sourced file calls `exit 1`, it exits **your entire current shell or calling script** (use `return 1` inside sourced files/functions instead).

## Examples with Explanations

### Reload your shell config without logging out

```bash
source ~/.bashrc
# POSIX shorthand:
. ~/.profile
```

### Load a library file relative to the script's directory

```bash
SCRIPT_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/lib/logging.sh"
```

Works reliably even when the script is invoked from a different working directory.

### Load a simple `KEY=VALUE` `.env` file

```bash
set -a
source ./.env
set +a
```

`set -a` (`allexport`) automatically exports every variable defined while sourcing `.env`, then `set +a` turns auto-export back off.

## Notes & Pitfalls

- **`source` is a Bash/Zsh keyword; `.` is POSIX-portable** (`/bin/sh` only supports `.`).
- If `filename` has no `/` (e.g., `source config.sh`), Bash searches `$PATH` first and then the current directory (unless `shopt -u sourcepath` is set)—always use `./config.sh` or an absolute path to avoid surprises.
- A sourced file does **not** need execute (`+x`) permission; it only needs read (`+r`) permission.

## Related Commands

- `export` — mark variables for child processes
- `exec` — replace the current shell process with another command
- `type` — inspect loaded functions and aliases

## Additional Resources

- `help source` / `help .` (in Bash)
