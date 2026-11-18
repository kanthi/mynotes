# fail2ban-client

## Overview

`fail2ban-client` controls and queries **Fail2ban**, a daemon that watches service logs (`sshd`, `nginx`, `postfix`) for repeated failed authentication or abuse patterns and automatically bans offending IP addresses via firewall rules (`nftables`, `iptables`, or `ufw`).

```bash
sudo apt install fail2ban   # Debian/Ubuntu
sudo dnf install fail2ban   # Fedora/RHEL
```

## Syntax

```bash
sudo fail2ban-client <command> [jail] [args...]
```

## Common Commands

| Command | Description |
|---------|-------------|
| `status` | List all active jails |
| `status <jail>` | Show failed attempts, total bans, and currently banned IPs for `<jail>` |
| `set <jail> unbanip <IP>` | Immediately unban a mistakenly blocked IP |
| `set <jail> banip <IP>` | Manually ban an IP in a jail |
| `reload` | Reload Fail2ban configuration without dropping existing bans |
| `get <jail> bantime` | Check configured ban duration for a jail |

## Key Use Cases

1. Check whether brute-force SSH attempts are being blocked (`fail2ban-client status sshd`)
2. Unban a colleague or CI runner that triggered too many failed logins (`unbanip`)
3. Reload rules safely after editing `/etc/fail2ban/jail.local`

## Safety

- Never edit `/etc/fail2ban/jail.conf` directly (package upgrades overwrite it)—put overrides in `/etc/fail2ban/jail.local` or `/etc/fail2ban/jail.d/*.conf`.
- Always whitelist your office/VPN/bastion IPs in `ignoreip` so you don't lock yourself out.

## Examples with Explanations

### Check all active jails and inspect `sshd`

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

Displays `Currently failed`, `Total failed`, `Currently banned`, and the `Banned IP list`.

### Unban a locked-out IP address

```bash
sudo fail2ban-client set sshd unbanip 203.0.113.42
```

Removes the firewall drop rule immediately without restarting Fail2ban.

### Reload configuration cleanly

```bash
sudo fail2ban-client reload
```

## Notes & Pitfalls

- On modern systemd distros without `/var/log/auth.log` (e.g., Debian 12+ or Fedora), set `backend = systemd` in `jail.local` so Fail2ban reads from `journald`.
- Fail2ban protects against noisy credential-stuffing, but **key-only SSH (`PasswordAuthentication no`)** is still your primary defense.

## Related Commands

- `nft` / `ufw` / `firewall-cmd` — underlying host firewall backends
- `journalctl` / `ausearch` — inspect failed SSH login attempts

## Additional Resources

- `man fail2ban-client`
- `man jail.conf`
