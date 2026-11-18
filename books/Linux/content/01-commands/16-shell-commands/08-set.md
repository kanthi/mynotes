# set

## Overview

`set` is a POSIX/Bash shell builtin that configures **shell execution options** (like strict error handling) and positional parameters (`$1`, `$2`, ...). In modern shell scripting, `set -euo pipefail` is the foundational safety header that prevents scripts from silently continuing after failed commands or typos in variable names.

## Syntax

```bash
set [-abefhkmnptuvxBCEHPT] [-o option-name] [--] [arg...]
set +o option-name   # disable an option
```

## Common Options

| Option | Long Form | Description |
|--------|-----------|-------------|
| `-e` | `errexit` | Exit immediately if a command exits with a non-zero status |
| `-u` | `nounset` | Treat unset variables and parameters as an error |
| `-o pipefail` | `pipefail` | Return the exit status of the last command in a pipeline that failed (not just the last command) |
| `-x` | `xtrace` | Print each command and its expanded arguments to stderr before executing (debugging) |
| `-C` | `noclobber` | Prevent `>` redirection from overwriting existing files (`>|` overrides) |
| `--` | — | End option parsing and assign remaining arguments to `$1`, `$2`, ... |

## Key Use Cases

1. Enable strict mode at the top of Bash scripts (`set -euo pipefail`)
2. Debug failing scripts line by line (`set -x` / `bash -x script.sh`)
3. Safely handle expected non-zero exits (like `grep` finding no match) without tripping `-e`

## Examples with Explanations

### Standard Bash strict mode header

```bash
#!/usr/bin/env bash
set -euo pipefail
```

Stops on unhandled errors (`-e`), catches unset variables (`-u`), and surfaces failures inside pipelines (`-o pipefail`).

### Temporarily allow a command to fail under `set -e`

```bash
set -euo pipefail

# grep returns 1 when no lines match; guard it with || true or an if-block
count=$(grep -c 'ERROR' app.log || true)
```

### Debug a specific section of a script

```bash
set -x
curl -fsS "https://api.example.com/health"
set +x
```

Turns on trace output only around the failing block, then disables it with `set +x`.

### Reset positional parameters safely

```bash
set -- "first arg" "second arg"
echo "$1"   # first arg
```

`--` ensures arguments starting with `-` are not misinterpreted as `set` flags.

## Notes & Pitfalls

- `set -e` does **not** exit on failures inside `if` conditions, `while` conditions, `||`/`&&` lists, or subshells whose exit status is checked—and inside functions called in an `if` condition, `-e` is ignored by Bash!
- `set -u` treats an empty array (`"${arr[@]}"` in Bash < 4.4) or unset optional `$1` as an error—use default expansion `"${1:-default}"`.
- Never put `set -euo pipefail` in your interactive `~/.bashrc`—a single failed `grep` or Ctrl+C will close your terminal session.

## Related Commands

- `trap` — clean up temp files or log errors on `EXIT` / `ERR`
- `shopt` — Bash-specific shell options (`nullglob`, `globstar`)
- `env` / `export` — inspect and export environment variables

## Additional Resources

- `help set` (in Bash)
- `man bash` (search `SHELL BUILTIN COMMANDS` -> `set`)
