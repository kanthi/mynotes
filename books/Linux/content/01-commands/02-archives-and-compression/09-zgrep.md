# zgrep

## Overview

`zgrep` runs `grep` over **gzip-compressed** files without manual decompression. It is a wrapper script around `gzip -dc` and `grep`, making it ideal for searching rotated system and web logs (`*.gz`) directly in place. Sibling tools exist for other formats: `xzgrep` (`.xz`), `bzgrep` (`.bz2`), and `zstdgrep` (`.zst`).

## Syntax

```bash
zgrep [grep-options] pattern [file...]
```

## Common Options

`zgrep` forwards standard `grep` flags:

| Option | Description |
|--------|-------------|
| `-i` | Case-insensitive match |
| `-E` | Extended regular expressions (`egrep` style) |
| `-F` | Fixed-string literal match (faster, no regex escaping needed) |
| `-H` | Always print the filename for each match (essential with multiple `.gz` files) |
| `-n` | Print line number within the decompressed stream |
| `-c` | Print only a count of matching lines per file |
| `-l` | Print only filenames that contain at least one match |
| `-v` | Invert match (show lines that do not match) |
| `-A N` / `-B N` / `-C N` | Show `N` lines After, Before, or Context around each match |

## Key Use Cases

1. Search rotated `/var/log/*.gz` files for errors, IPs, or request IDs
2. Find which archived log contains a specific trace ID before inspecting it
3. Count error occurrences across days of compressed logs
4. Combine with `find` and `xargs` to search nested archive trees

## Examples with Explanations

### Search rotated logs with filenames and line numbers

```bash
zgrep -Hn 'OutOfMemoryError' /var/log/app/app.log.*.gz
```

`-H` ensures you know which day's rotated file matched, even if only one file is passed.

### Match HTTP 5xx responses with extended regex

```bash
zgrep -E 'HTTP/1\.[01]" 5[0-9]{2}' /var/log/nginx/access.log.*.gz
```

Uses `-E` for alternation/quantifiers without needing backslash-escaped braces.

### Find which compressed logs mention a trace ID

```bash
zgrep -l 'req-9f8a21c' /var/log/api/*.gz
```

Stops reading each file after the first match and prints only the matching filenames.

### Show surrounding context lines around an error

```bash
zgrep -C 5 'panic:' /var/log/worker.log.1.gz
```

Prints 5 lines before and 5 lines after each `panic:` so you can see the stack trace or preceding state.

### Search mixed compressed and uncompressed logs together

```bash
zgrep -H 'Failed password' /var/log/auth.log /var/log/auth.log.*.gz
```

`zgrep` handles both plain text and `.gz` files in the same invocation.

### Parallel search across a large log tree

```bash
find /var/log/archive -name '*.gz' -print0 | xargs -0 -P 4 zgrep -Hn 'timeout'
```

Runs up to 4 `zgrep` workers in parallel when scanning hundreds of rotated files.

### Count matches per archive

```bash
zgrep -c 'ERROR' /var/log/app/*.gz
```

## Notes & Pitfalls

- `zgrep` decompresses files in memory on every run, which is much slower than `rg` or `grep` on plain text—decompress hot logs if you query them repeatedly.
- Multiple-file searches do not support `grep -r` recursion cleanly across `.gz` files; use `find ... -print0 | xargs -0 zgrep` instead.
- If a file inside `.gz` contains binary bytes, `grep` may stop with `Binary file matches`; pass `-a` to treat it as text or `-I` to skip binary streams.
- `ripgrep` (`rg -z`) can also search `.gz`, `.xz`, `.bz2`, and `.zst` files natively with multi-core speed when installed.

## Related Commands

- `grep` / `rg` — search plain text (and `rg -z` for compressed files)
- `zcat` / `zless` — stream or page through `.gz` files
- `xzgrep` / `bzgrep` / `zstdgrep` — equivalents for `.xz`, `.bz2`, and `.zst`
- `journalctl` — query structured systemd logs directly

## Additional Resources

- `man zgrep`
- `man grep`
