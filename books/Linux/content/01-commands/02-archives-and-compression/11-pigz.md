# pigz

## Overview

`pigz` (**Parallel Implementation of GZip**) is a drop-in, multi-core replacement for `gzip`. Classic `gzip` runs single-threaded and bottlenecks on one CPU core during large database dumps or `tar` backups; `pigz` splits compression across all available CPU cores while producing standard `.gz` files readable by `gzip`, `zcat`, and `zgrep`.

```bash
sudo apt install pigz   # Debian/Ubuntu
sudo dnf install pigz   # Fedora/RHEL
```

## Syntax

```bash
pigz [options] [file...]
unpigz [options] [file...]
```

## Common Options

| Option | Description |
|--------|-------------|
| `-p N` | Use `N` compression threads (default: all online CPU cores) |
| `-k`, `--keep` | Keep the original input file (unlike default `gzip`/`pigz` behavior) |
| `-d`, `--decompress` | Decompress `.gz` files (or invoke `unpigz`) |
| `-c`, `--stdout` | Write compressed/decompressed output to stdout |
| `-1` .. `-9` | Compression level (`-1` fastest, `-6` default, `-9` smallest) |
| `-11` | Zopfli-style extra-tight compression (very slow, smaller `.gz` for static assets) |
| `-t`, `--test` | Test `.gz` archive integrity |

## Key Use Cases

1. Speed up `tar` backups of large directories (`tar -I pigz -cf ...`) on multi-core servers
2. Compress multi-gigabyte SQL dumps in a fraction of the time (`mysqldump | pigz`)
3. Limit threads (`-p 4`) on production hosts so backups don't saturate all CPU cores
4. Decompress `.gz` files faster with `unpigz`

## Safety

- Just like `gzip`, `pigz file.log` **removes the original `file.log`** unless you pass `-k` (`--keep`) or `-c`.
- On busy production servers, cap threads with `-p 2` or `-p 4` (and pair with `nice -n 19`) so backup compression doesn't starve application threads.

## Examples with Explanations

### Compress a `tar` archive across all CPU cores

```bash
tar -I pigz -cf backup.tar.gz /var/www
```

`-I pigz` tells GNU `tar` to use `pigz` instead of single-threaded `gzip`.

### Cap threads on a production server

```bash
tar -I "pigz -p 4 -6" -cf app-logs.tar.gz /var/log/app
```

Limits `pigz` to 4 threads at default compression level `-6`.

### Speed up a PostgreSQL or MySQL dump

```bash
pg_dump -U postgres appdb | pigz -p 4 > appdb.sql.gz
```

### Keep the original file while compressing

```bash
pigz -k -p 4 large-export.csv
```

Produces `large-export.csv.gz` without deleting `large-export.csv`.

### Decompress and verify integrity

```bash
unpigz -t appdb.sql.gz
unpigz -k appdb.sql.gz
```

*(Note: decompression itself is mostly single-threaded due to DEFLATE stream constraints, though `unpigz` parallelizes checksums and I/O.)*

### Pipe with `pv` to watch parallel compression speed

```bash
tar -cf - /srv/data | pv | pigz -p 8 > data.tar.gz
```

## Notes & Pitfalls

- If you do not need `.gz` compatibility with legacy tools, **`zstd` (`zstd -T0`)** usually beats `pigz` in both compression speed and compression ratio.
- `pigz` writes standard `.gz` streams, so `zcat`, `zgrep`, and `gunzip` on any host can read files created by `pigz`.

## Related Commands

- `gzip` — classic single-threaded `.gz` compressor
- `zstd` — modern multi-threaded compressor (`-T0`) with better speed/ratio
- `tar` — archive trees with `-I pigz`
- `pv` — monitor pipeline throughput

## Additional Resources

- `man pigz`
