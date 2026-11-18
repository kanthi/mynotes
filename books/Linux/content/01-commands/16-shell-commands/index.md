---
title: Intro
---

# Intro

Everyday shell built-ins and small utilities for output, history, name resolution, environment, timing, directory stacks, and reliable scripting (`set`, `trap`, `read`, `exec`, `source`, `getopts`). Deeper programming belongs in **Scripting → Bash** (this book).

## Commands in this part

| Command | Role |
|---------|------|
| `echo` | echo writes its arguments to standard output, separated by spaces, usually followed by a newline. |
| `printf` | printf formats and prints arguments using a format string, like C’s printf. |
| `history` | history shows the shell’s command history list (bash). |
| type / command | type explains how the shell will resolve a name — alias, keyword, function, builtin, or external file. |
| `export` | export marks shell variables for the environment so child processes inherit them. |
| `time` | time runs a command and reports how long it took. |
| pushd / popd / dirs | pushd, popd, and dirs maintain a directory stack in bash. |
| `set` | set configures shell execution options (such as `set -euo pipefail` strict mode and `-x` tracing) and positional parameters. |
| `trap` | trap runs cleanup or error-handling commands when a script exits or receives a signal (`EXIT`, `ERR`, `INT`, `TERM`). |
| `read` | read safely reads lines, delimited fields, or interactive/secret input into shell variables. |
| `exec` | exec replaces the shell process with a command (container `ENTRYPOINT` PID 1 pattern) or redirects script file descriptors. |
| `source` / `.` | source executes a file inside the current shell environment to load `.env` files, configs, or helper functions. |
| `getopts` | getopts parses short CLI flags (`-v`, `-c file`) and options inside shell scripts and functions. |

## Suggested starting points

1. Print: `echo`, `printf`.
2. Recall & inspect: `history`, `type` / `command`.
3. Environment & sourcing: `export`, `source` / `.` (see also `env` in System information).
4. Safe scripting foundations: `set` (`set -euo pipefail`), `trap`, `read`, `getopts`, `exec`.
5. Measure & navigate: `time`, `pushd`/`popd`/`dirs`.

## Related parts

- Help and documentation tools — `help` for builtins
- Text and pipes — consume what you print
- Scripting → Bash (this book) — functions, arrays, strict mode

Continue with the individual command pages in this part.
