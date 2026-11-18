# sftp

## Overview

`sftp` is the interactive **SSH File Transfer Protocol** client. It runs over an encrypted SSH channel (default port 22), shares `~/.ssh/config` and `ssh-agent` authentication, and provides an FTP-like shell for browsing, uploading, and downloading files—even on restricted accounts (`internal-sftp`) where interactive shell login is disabled.

## Syntax

```bash
sftp [options] [user@]host[:path]
sftp [options] -b batchfile [user@]host
```

## Common Options & Interactive Commands

| Option / Command | Description |
|------------------|-------------|
| `-i KEY` | Private key identity file |
| `-P PORT` | Remote SSH port (**capital `-P`**, unlike `ssh -p`) |
| `-o OPT=VAL` | Pass SSH config options (e.g., `ProxyJump=bastion`) |
| `-b FILE` | Run commands non-interactively from a batch file (`-` for stdin) |
| `-r` | Recursively copy entire directories |
| `-a` | Attempt to resume interrupted transfers |
| `-C` | Enable SSH compression |
| `get` / `put` | *(Interactive)* Download / upload files (`-r` recursive, `-p` preserve permissions/times) |
| `ls` / `lls` | *(Interactive)* List remote (`ls`) or local (`lls`) directory |
| `cd` / `lcd` | *(Interactive)* Change remote (`cd`) or local (`lcd`) directory |

## Key Use Cases

1. Browse and transfer files on SFTP-only / chrooted vendor servers where `ssh` shell access is blocked
2. Resume large interrupted downloads (`get -a`)
3. Automate deterministic file drops with a batch script (`-b`)
4. Transfer files through a bastion jump host (`-o ProxyJump=`)

## Examples with Explanations

### Interactive session with local/remote navigation

```bash
sftp -i ~/.ssh/id_ed25519 -P 2222 alice@server.example.com
# sftp> lls
# sftp> cd /var/backups
# sftp> get -p db-2026-11-18.sql.zst
# sftp> put -r ./dist /var/www/app/
# sftp> bye
```

Prefixing local commands with `l` (`lls`, `lcd`, `lpwd`) lets you inspect local files without leaving the SFTP prompt.

### One-line non-interactive download

```bash
sftp alice@server.example.com:/var/log/nginx/error.log ./
```

Downloads a single file directly without entering interactive mode.

### Automated batch transfer

```bash
sftp -b - alice@server.example.com <<'EOF'
cd /var/backups
lcd /tmp/restore
get -r weekly/
bye
EOF
```

### Connect through a jump host

```bash
sftp -o ProxyJump=bastion.example.com alice@10.0.1.50
```

## Notes & Pitfalls

- `sftp` and `scp` use **capital `-P`** for port, whereas `ssh` uses lowercase `-p`.
- For syncing large directory trees or resuming delta transfers repeatedly, `rsync -avz -e ssh` is much faster than `sftp`.
- In batch mode (`-b`), `sftp` aborts immediately on the first failed command unless the line is prefixed with `-` (e.g., `-rm old_file.txt`).

## Related Commands

- `rsync` — fast incremental file sync over SSH
- `scp` — quick one-off remote copy
- `ssh` — interactive remote shell
- `ssh-keygen` — manage SSH keys

## Additional Resources

- `man sftp`
- `man sftp-server`
