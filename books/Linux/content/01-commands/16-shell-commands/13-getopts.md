# getopts

## Overview

`getopts` is the standard POSIX/Bash shell builtin for **parsing short command-line flags** (`-v`, `-f file`, `-n 10`, or combined `-vf file`) inside scripts and shell functions. It handles option bundling, missing option arguments, and unknown flags cleanly without external dependencies.

## Syntax

```bash
getopts optstring name [arg...]
```

- Leading `:` in `optstring` (e.g., `:f:v`) enables **silent error reporting** so your script can print custom error messages.
- Trailing `:` after a letter (e.g., `f:`) means that option **requires an argument** (stored in `$OPTARG`).

## Key Variables

| Variable | Meaning |
|----------|---------|
| `$OPTARG` | Argument value for the current option (or the invalid option character in silent mode) |
| `$OPTIND` | Index of the next argument to process (starts at `1`; reset to `1` when reusing in functions) |

## Key Use Cases

1. Give Bash scripts clean `-h`, `-v`, `-f FILE`, and `-n COUNT` CLI flags
2. Support bundled flags like `-vh` automatically
3. Separate flags from trailing positional arguments using `shift $((OPTIND - 1))`

## Examples with Explanations

### Complete CLI flag parser template

```bash
#!/usr/bin/env bash
set -euo pipefail

verbose=0
config=""

usage() {
  echo "Usage: $0 [-v] [-c config_file] target..." >&2
  exit 2
}

while getopts ":vc:h" opt; do
  case "$opt" in
    v) verbose=1 ;;
    c) config="$OPTARG" ;;
    h) usage ;;
    :) echo "Error: -$OPTARG requires an argument." >&2; usage ;;
    \?) echo "Error: Unknown option -$OPTARG" >&2; usage ;;
  esac
done

shift $((OPTIND - 1))
echo "verbose=$verbose config=$config remaining_args=$*"
```

`shift $((OPTIND - 1))` removes all parsed flags so `$1`, `$2`, and `$@` now refer only to the positional arguments after the flags.

### Reset `OPTIND` inside reusable shell functions

```bash
myfunc() {
  local OPTIND=1 opt
  while getopts "ab:" opt; do
    ...
  done
}
```

Making `OPTIND` local and setting it to `1` prevents state from leaking across multiple function calls.

## Notes & Pitfalls

- `getopts` only supports **short single-letter options** (`-v`, `-c file`), not GNU long options (`--verbose`, `--config=file`). If you need long options, use `getopt` (util-linux `getopt`, not the builtin) or a manual `while [[ $# -gt 0 ]]` `case` loop.
- `getopts` stops parsing at the first non-option argument (or `--`), so flags must come before positional arguments.

## Related Commands

- `set` — `set --` and strict mode
- `read` — interactive prompts

## Additional Resources

- `help getopts` (in Bash)
