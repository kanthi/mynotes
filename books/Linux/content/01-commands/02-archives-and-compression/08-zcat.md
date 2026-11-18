# zcat

## Overview

`zcat` writes **gzip-compressed** files to stdout as if they were uncompressed (equivalent to `gzip -dc`). It lets you inspect, filter, and stream `.gz` archives or rotated logs in pipelines without creating temporary uncompressed files on disk. For `.xz`, `.bz2`, or `.zst` files, use `xzcat`, `bzcat`, or `zstdcat` respectively.

## Syntax

```bash
zcat [options] [file...]
gzip -dc [file...]
```

## Common Options

| Option | Description |
|--------|-------------|
| `-f`, `--force` | Force decompression even if the file has multiple links or is not `.gz` (passes uncompressed data through) |
| `-l`, `--list` | Display compressed/uncompressed sizes and ratio instead of writing content |
| `-t`, `--test` | Test compressed file integrity without outputting data |
| `-v`, `--verbose` | Print name and percentage reduction (useful with `-t` or `-l`) |
| `-q`, `--quiet` | Suppress warnings |

## Key Use Cases

1. Inspect rotated logs (`.log.1.gz`, `.log.2.gz`) without unpacking them to disk
2. Stream compressed SQL dumps or CSVs directly into `mysql`, `psql`, `jq`, or `awk`
3. Concatenate multiple gzipped shards into a single stream
4. Verify archive integrity before deleting or archiving backups

## Examples with Explanations

### Read a rotated log with a pager

```bash
zcat /var/log/nginx/access.log.2.gz | less
zcat /var/log/syslog.1.gz | grep -i 'oom' | tail -n 20
```

Streams directly into `less` or `grep` without touching `/tmp` or your current directory.

### Stream a compressed database dump into PostgreSQL or MySQL

```bash
zcat backup-2026-11-18.sql.gz | psql -U postgres appdb
zcat dump.sql.gz | mysql -u root -p appdb
```

Avoids needing double the disk space for an intermediate `.sql` file.

### Concatenate multiple compressed shards

```bash
zcat events-part-*.jsonl.gz | jq -c 'select(.status >= 500)' > errors.jsonl
```

Decompresses each shard in order into a single continuous stream for `jq`.

### Check uncompressed size before extracting

```bash
zcat -l large-archive.tar.gz
```

Shows compressed size, uncompressed size, compression ratio, and original filename so you do not run out of disk space.

### Verify integrity without writing output

```bash
zcat -t backup.sql.gz && echo "Archive OK"
```

Exits `0` if valid, non-zero if truncated or corrupted.

### Portable script alternative

```bash
gzip -dc file.gz | head -n 50
```

On some non-Linux or minimal systems (such as macOS, where `zcat` expects `.Z` compress files unless using `gzcat`), `gzip -dc` is completely portable.

## Understanding Output

With `-l`, `zcat` prints four columns:

| Column | Meaning |
|--------|---------|
| `compressed` | Size of the `.gz` file in bytes |
| `uncompressed` | Original size in bytes (modulo 4 GiB in gzip headers for files over 4 GiB) |
| `ratio` | Space savings percentage |
| `uncompressed_name` | Stored original filename |

## Notes & Pitfalls

- `zcat` only handles gzip (and legacy `compress` `.Z`) files; passing `.xz` or `.zst` fails unless your distro ships a wrapper.
- Gzip stores uncompressed size as a 32-bit integer, so `zcat -l` wraps around for files larger than 4 GiB.
- Redirecting `zcat huge.gz > huge.txt` expands the full file on disk—keep it in a pipeline (`|`) whenever possible.
- For quick pattern matching across many `.gz` files, `zgrep` is cleaner than `zcat ... | grep` because `zgrep -H` preserves filenames.

## Related Commands

- `gzip` / `gunzip` — compress and decompress `.gz` files in place
- `zgrep` / `zless` — search or page through `.gz` files directly
- `xzcat` / `bzcat` / `zstdcat` — stream-decompress `.xz`, `.bz2`, and `.zst` files
- `tar -xzOf` — extract a single member of a `.tar.gz` archive to stdout

## Additional Resources

- `man zcat`
- `man gzip`
