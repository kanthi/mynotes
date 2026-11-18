# read

## Overview

`read` is a shell builtin that reads a single line from standard input (or a file descriptor) and splits it into one or more shell variables. It is used for safe line-by-line file processing, parsing delimited records (`IFS=:`), and interactive CLI prompts (including hidden password prompts).

## Syntax

```bash
read [options] [var1 var2 ...]
```

## Common Options

| Option | Description |
|--------|-------------|
| `-r` | **Raw mode** — do not treat backslashes (`\`) as escape characters (always use `-r`!) |
| `-p "prompt"` | Print a prompt string to stderr before reading (interactive shells) |
| `-s` | Silent mode — do not echo typed characters (for passwords/tokens) |
| `-t N` | Timeout after `N` seconds (returns non-zero if timed out) |
| `-n N` | Read at most `N` characters instead of waiting for newline |
| `-a arr` | Read words into an indexed Bash array `arr` |
| `-d delim` | Stop at custom delimiter `delim` instead of newline (use `-d ''` for NUL-delimited input) |

## Key Use Cases

1. Iterate safely over lines in a file or command output (`while IFS= read -r line`)
2. Process filenames containing spaces or newlines from `find -print0` (`IFS= read -r -d ''`)
3. Prompt users for confirmation or secrets in setup scripts (`read -rsp`)

## Examples with Explanations

### Safe line-by-line file loop (gold standard)

```bash
while IFS= read -r line; do
  printf 'Line: %s\n' "$line"
done < /etc/hosts
```

`IFS=` prevents trimming leading/trailing whitespace, and `-r` preserves backslashes verbatim.

### Safely loop over `find -print0` output (handles spaces & newlines)

```bash
while IFS= read -r -d '' file; do
  echo "Processing: $file"
done < <(find /var/log -name '*.log' -print0)
```

Using process substitution `< <(...)` keeps the loop in the **current** shell instead of a subshell, so variables modified inside the loop persist afterward.

### Split colon-delimited fields (e.g., `/etc/passwd`)

```bash
while IFS=: read -r user _ uid gid _ home shell; do
  echo "$user ($uid) -> $shell"
done < /etc/passwd
```

`_` discards fields you do not care about.

### Interactive confirmation and secret prompt

```bash
read -r -p "Continue deployment? [y/N] " answer
[[ "${answer,,}" == "y" ]] || exit 1

read -r -s -p "API token: " token
echo
```

## Notes & Pitfalls

- **Never pipe into `while read`** (`cmd | while read ...`) if you set variables inside the loop—the pipe runs the `while` loop in a subshell, and variable changes vanish when the loop ends. Use `< <(cmd)` instead.
- If the last line of a file has no trailing newline, `read` reads it **and returns exit status 1**, causing `while IFS= read -r line` to skip it! Fix with: `while IFS= read -r line || [[ -n "$line" ]]; do ...`.
- Always pass `-r` unless you intentionally want backslash line-continuation.

## Related Commands

- `printf` — safe formatted output
- `xargs` — build command invocations from stdin
- `awk` / `cut` — high-speed field extraction on large files

## Additional Resources

- `help read` (in Bash)
