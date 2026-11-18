# nslookup

## Overview

`nslookup` queries DNS name servers interactively or non-interactively to resolve hostnames, IP addresses, and DNS records (`A`, `AAAA`, `MX`, `TXT`, `NS`, `PTR`). While it remains common in legacy runbooks and cross-platform Windows/Linux troubleshooting, modern Linux workflows generally prefer **`dig`**, **`host`**, **`doggo`**, or **`resolvectl query`** for cleaner output and reliable scripting.

```bash
sudo apt install dnsutils     # Debian/Ubuntu
sudo dnf install bind-utils   # Fedora/RHEL
```

## Syntax

```bash
nslookup [options] [name | -] [server]
```

## Common Options & Interactive Directives

| Option / Directive | Description |
|--------------------|-------------|
| `-type=RECORD` | Query record type (`a`, `aaaa`, `mx`, `ns`, `soa`, `txt`, `ptr`, `cname`) |
| `-debug` | Print full DNS packet details, flags, and TTLs |
| `-port=N` | Query a non-standard DNS port |
| `-timeout=N` | Timeout in seconds per retry |
| `server ADDR` | *(Interactive)* Switch the target DNS resolver |
| `set type=mx` | *(Interactive)* Set record type for subsequent queries |
| `set debug` | *(Interactive)* Toggle packet debug output |

## Key Use Cases

1. Quickly check `A`, `MX`, or `TXT` records from a minimal host or legacy runbook
2. Test how a specific DNS server (`1.1.1.1`, `8.8.8.8`, or internal DNS) answers a query
3. Run multiple consecutive queries against different resolvers in interactive mode
4. Perform quick reverse DNS (`PTR`) lookups on an IP address

## Examples with Explanations

### Standard forward and reverse lookups

```bash
nslookup example.com
nslookup 1.1.1.1
```

Resolves `A`/`AAAA` records using `/etc/resolv.conf`, or queries the `PTR` record for an IP.

### Query specific record types (`MX`, `TXT`, `NS`)

```bash
nslookup -type=mx example.com
nslookup -type=txt example.com
nslookup -type=ns example.com
```

### Query a specific DNS resolver directly

```bash
nslookup example.com 1.1.1.1
```

Bypasses the local system stub resolver to compare what an upstream resolver returns.

### Inspect TTL and response flags with `-debug`

```bash
nslookup -debug example.com
```

### Interactive mode for multiple queries

```bash
nslookup
> server 8.8.8.8
> set type=txt
> example.com
> exit
```

### Modern equivalents (preferred for scripts and clarity)

```bash
dig +short example.com @1.1.1.1
host -t MX example.com
resolvectl query example.com
doggo example.com MX
```

## Notes & Pitfalls

- **Never parse `nslookup` output in scripts**—its formatting varies across `bind-utils` versions and OSes; use `dig +short` or `doggo --json`.
- `nslookup` ignores `/etc/hosts` and nsswitch—it queries DNS directly, which can differ from what `ping` or `curl` resolves (use `getent hosts` to test full system resolution).
- `ANY` queries are blocked or truncated by many public resolvers (such as Cloudflare `1.1.1.1`).

## Related Commands

- `dig` — detailed, script-friendly DNS client
- `host` — concise DNS lookup utility
- `doggo` — modern DNS client with JSON and DoH/DoT support
- `resolvectl` — inspect and query `systemd-resolved`
- `getent` — test full `/etc/nsswitch.conf` resolution (including `/etc/hosts`)

## Additional Resources

- `man nslookup`
