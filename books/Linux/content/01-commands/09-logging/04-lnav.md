# lnav

## Overview

`lnav` (Log File Navigator) is an interactive terminal log viewer that automatically detects log formats (syslog, Nginx, Apache, JSON lines, systemd journal exports, and compressed `.gz`/`.bz2` archives), merges multiple files chronologically into a single view, and lets you filter or query logs using **SQL** without setting up a centralized log stack.

```bash
sudo apt install lnav   # Debian/Ubuntu
sudo dnf install lnav   # Fedora/RHEL
```

## Syntax

```bash
lnav [options] [file_or_dir...]
```

## Common Options & Interactive Keys

| Flag / Key | Description |
|------------|-------------|
| `-r` | Load rotated log files too (e.g., `access.log`, `access.log.1`, `access.log.2.gz`) |
| `-t` | Prepend timestamps to piped stdin lines |
| `-n` | Run headless (non-interactive) and execute `-c` commands |
| `-c ';SELECT ...'` | Run an SQL query or lnav command at startup |
| `e` / `E` | Jump to next / previous error |
| `w` / `W` | Jump to next / previous warning |
| `:` | Open command prompt (`:filter-in`, `:filter-out`, `:hide-lines-before`) |
| `;` | Open SQLite query prompt over parsed log fields |
| `/` | Regex search |
| `p` | Toggle pretty-print view for JSON/XML log lines |

## Key Use Cases

1. Merge and correlate `/var/log/syslog`, `/var/log/nginx/error.log`, and app logs on one timeline
2. Read live logs alongside rotated `.gz` archives (`lnav -r`) without unpacking
3. Run ad-hoc SQL aggregations (top 5xx URLs, slowest requests, error counts by hour)
4. Pretty-print structured JSON logs in the terminal

## Examples with Explanations

### View and correlate multiple log files on a single timeline

```bash
sudo lnav /var/log/syslog /var/log/nginx/error.log /var/log/auth.log
```

`lnav` parses timestamps from each format and interleaves every line chronologically, color-coding `ERROR` and `WARN` levels.

### Include rotated and compressed logs automatically

```bash
sudo lnav -r /var/log/nginx/access.log
```

Loads `access.log`, `access.log.1`, `access.log.2.gz`, etc. together as one continuous stream.

### Pipe `journalctl` output into `lnav`

```bash
journalctl -u nginx -o json | lnav
```

Gives you `lnav`'s navigation (`e`/`E` for errors), search, and SQL querying over systemd journal JSON records.

### Run a non-interactive SQL query over Nginx logs

```bash
lnav -n -c ';SELECT c_ip, count(*) AS hits FROM access_log WHERE sc_status >= 500 GROUP BY c_ip ORDER BY hits DESC LIMIT 10' /var/log/nginx/access.log
```

Uses `lnav`'s built-in SQLite engine to aggregate 5xx errors by client IP without writing `awk` or Python scripts.

### Filter noise interactively

Inside `lnav`, press `:` and run:

```text
:filter-out healthcheck
:filter-in timeout|502
:hide-lines-before 2026-11-18 14:00:00
```

Hides health-check spam and focuses strictly on the incident window.

## Notes & Pitfalls

- `lnav` indexes files in memory—pointing `-r` at gigabytes of historical archives can take time and RAM; narrow to the relevant directory or glob when possible.
- Custom application log formats work out of the box if they emit JSON lines; for custom text formats, add a format JSON under `~/.lnav/formats/installed/`.
- Press `?` inside `lnav` at any time for a built-in interactive cheat sheet.

## Related Commands

- `dmesg` — kernel ring buffer
- `journalctl` — systemd journal query tool
- `zgrep` / `zcat` — search or stream compressed `.gz` logs
- `jq` — filter JSON logs in pipelines

## Additional Resources

- `man lnav`
- Official documentation: `https://docs.lnav.org/`
