# ssh-copy-id

## Overview

`ssh-copy-id` installs your local SSH public key into a remote user's `~/.ssh/authorized_keys` file, automatically creating `~/.ssh`, setting strict `0700`/`0600` permissions, avoiding duplicate entries, and restoring SELinux contexts where supported. Always prefer it over manual `echo >>` pipelines when bootstrapping password-less SSH access.

## Syntax

```bash
ssh-copy-id [options] [user@]host
```

## Common Options

| Option | Description |
|--------|-------------|
| `-i KEY.pub` | Specify which public key to install (defaults to your default/agent keys) |
| `-p PORT` | Connect to a non-standard SSH port |
| `-o OPT=VAL` | Pass underlying `ssh` options (e.g., `ProxyJump`, `StrictHostKeyChecking`) |
| `-n` | Dry-run—print the key(s) that would be installed without modifying the remote host |
| `-f` | Force mode—add the key without checking if it is already present |
| `-s` | Use SFTP instead of shell execution (useful for restricted accounts) |

## Key Use Cases

1. Bootstrap key-based SSH login on a newly provisioned server
2. Install a specific key (`-i`) onto a host using a non-standard port (`-p`)
3. Push a public key to an internal host behind a bastion (`-o ProxyJump=`)
4. Preview which keys `ssh-copy-id` plans to copy (`-n`) before touching the server

## Safety

- `ssh-copy-id` authenticates with your existing remote password (or an already-loaded key); once verified, consider disabling `PasswordAuthentication` in `/etc/ssh/sshd_config`.
- If you omit `-i` while `ssh-agent` holds multiple keys, `ssh-copy-id` may install **all** agent keys unless you pass `-i ~/.ssh/specific_key.pub`.
- OpenSSH refuses key auth if the remote home directory or `~/.ssh` is group/world-writable.

## Examples with Explanations

### Install your default public key and test login

```bash
ssh-copy-id alice@server.example.com
ssh alice@server.example.com
```

### Install a specific Ed25519 key on a custom port

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 2222 alice@server.example.com
```

### Preview which key(s) would be copied (dry-run)

```bash
ssh-copy-id -n -i ~/.ssh/id_ed25519.pub alice@server.example.com
```

### Copy a key to a private server via a bastion host

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub -o ProxyJump=bastion.example.com alice@10.0.1.50
```

### Manual fallback when `ssh-copy-id` is unavailable

```bash
cat ~/.ssh/id_ed25519.pub | ssh alice@server.example.com \
  'umask 077; mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys'
```

## Notes & Pitfalls

- If the server already disables password authentication, `ssh-copy-id` cannot log in—place the initial key via cloud-init, provider console, or an existing admin session.
- On SELinux-enforcing servers, if manual copying breaks key login, run `restorecon -Rv ~/.ssh` on the server.

## Related Commands

- `ssh-keygen` — generate and inspect SSH key pairs
- `ssh` — connect to remote hosts
- `restorecon` — fix SELinux labels on `~/.ssh/authorized_keys`
- `getent` — verify remote user home directory

## Additional Resources

- `man ssh-copy-id`
