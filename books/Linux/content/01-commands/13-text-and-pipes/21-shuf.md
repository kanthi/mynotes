# shuf

## Overview

`shuf` (part of GNU coreutils) outputs a **random permutation** of input lines, integer ranges, or command-line arguments. It is the cleanest, fastest way in Bash to sample N random lines from a massive log/CSV file, pick random items for testing, or generate random test numbers without writing Python or Perl one-liners.

## Syntax

```bash
shuf [options] [file]
shuf -e [options] [args...]
shuf -i LO-HI [options]
```

## Common Options

| Option | Description |
|--------|-------------|
| `-n COUNT` | Output at most `COUNT` lines (random sample) |
| `-i LO-HI` | Treat each integer from `LO` through `HI` inclusive as an input line |
| `-e`, `--echo` | Treat each command-line argument as an input line |
| `-r`, `--repeat` | Allow repeated output lines (sampling **with replacement**) |
| `-z`, `--zero-terminated` | Use NUL (`\0`) as line delimiter instead of newline |
| `-o FILE` | Write output to `FILE` (safe even when `FILE` is also the input) |

## Key Use Cases

1. Pull a random 100-line sample from a multi-gigabyte CSV or access log (`shuf -n 100`)
2. Pick a random port, host, or test case from a list (`shuf -n1 -e ...` or `shuf -i`)
3. Shuffle a playlist or test suite order (`shuf tests.txt`)
4. Generate random jitter/sleep delays in scripts (`sleep "$(shuf -i 1-30 -n 1)"`)

## Examples with Explanations

### Sample 20 random lines from a huge log file

```bash
shuf -n 20 /var/log/nginx/access.log
```

`shuf -n 20` uses reservoir sampling so it does not have to load or sort the whole file in memory first.

### Pick a random integer (e.g., random jitter or ephemeral port)

```bash
shuf -i 1-60 -n 1
sleep "$(shuf -i 5-30 -n 1)"
```

Great for adding random jitter to cron scripts so 100 servers don't hit an API at the exact same second.

### Pick one random item from command-line arguments (`-e`)

```bash
shuf -n 1 -e us-east-1 us-west-2 eu-west-1
```

### Generate a random 6-digit PIN or roll dice (`-r` with replacement)

```bash
shuf -r -n 5 -i 1-6
```

`-r` allows the same number to appear more than once; without `-r`, `shuf` outputs each item at most once.

### Shuffle a file in place safely (`-o`)

```bash
shuf -o hosts.txt hosts.txt
```

`-o` reads all input before opening `hosts.txt` for writing, avoiding the `> hosts.txt` truncation trap.

### Sample filenames containing spaces safely (`-z`)

```bash
find /data/images -type f -print0 | shuf -z -n 5 | xargs -0 ls -lh
```

## Notes & Pitfalls

- `shuf` uses a standard PRNG, **not** a cryptographic generator—never use `shuf` to generate passwords, tokens, or keys (use `openssl rand` or `/dev/urandom` instead).
- On macOS (BSD), `shuf` is not installed by default (`brew install coreutils` provides `gshuf`).

## Related Commands

- `sort` (`sort -R` also shuffles, though `shuf` is much faster and supports `-n`/`-i`)
- `head` / `tail` — take top/bottom lines deterministically
- `seq` — generate sequential number ranges

## Additional Resources

- `man shuf`
