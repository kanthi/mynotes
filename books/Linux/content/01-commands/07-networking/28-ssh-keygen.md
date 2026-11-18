# ssh-keygen

## Overview

`ssh-keygen` creates, inspects, converts, and manages **SSH key pairs** and `known_hosts` entries. On modern Linux systems, **Ed25519** (`-t ed25519`) is the recommended default for speed, short keys, and strong cryptography, while 4096-bit RSA remains available for legacy appliances.

## Syntax

```bash
ssh-keygen [options]
ssh-keygen -t ed25519 -C "comment" -f ~/.ssh/id_ed25519
```

## Common Options

| Option | Description |
|--------|-------------|
| `-t TYPE` | Key algorithm: `ed25519` (recommended), `ecdsa`, `rsa`, `ed25519-sk` (FIDO2 hardware key) |
| `-b BITS` | Bit length for RSA keys (use `4096`) |
| `-C "comment"` | Label appended to the public key (e.g., `user@laptop` or email) |
| `-f PATH` | Private key output/input path |
| `-N "pass"` | Set passphrase non-interactively (`""` for empty—use with care) |
| `-p` | Change the passphrase of an existing private key without regenerating it |
| `-l -f FILE` | Display key fingerprint and bit length |
| `-y -f KEY` | Derive and print the public key from a private key |
| `-F HOST` | Search `~/.ssh/known_hosts` for a host's fingerprint |
| `-R HOST` | Remove stale host key(s) from `~/.ssh/known_hosts` |
| `-A` | Generate any missing server host keys (admin use on `/etc/ssh/`) |

## Key Use Cases

1. Generate a new Ed25519 key pair for interactive logins or Git/CI authentication
2. Verify key fingerprints (`-lf`) before trusting a server or adding a deploy key
3. Clean up stale `known_hosts` entries (`-R`) after a server rebuild
4. Add or rotate a passphrase (`-p`) on an existing private key

## Safety

- Keep private keys at mode `0600` and `~/.ssh` at mode `0700`—OpenSSH refuses keys with looser permissions.
- Protect interactive keys with a strong passphrase and unlock them once per session via `ssh-agent`.
- For automated/CI keys without a passphrase, lock down the server's `~/.ssh/authorized_keys` line with `restrict,command="/path/to/script"` and `from="10.0.0.0/24"`.

## Examples with Explanations

### Generate a modern Ed25519 key pair

```bash
ssh-keygen -t ed25519 -C "alice@workstation" -f ~/.ssh/id_ed25519
```

Creates `~/.ssh/id_ed25519` (private) and `~/.ssh/id_ed25519.pub` (public).

### Generate a 4096-bit RSA key for legacy systems

```bash
ssh-keygen -t rsa -b 4096 -C "legacy-appliance" -f ~/.ssh/id_rsa_legacy
```

### Inspect key fingerprints (SHA256 and legacy MD5)

```bash
ssh-keygen -lf ~/.ssh/id_ed25519.pub
ssh-keygen -E md5 -lf ~/.ssh/id_ed25519.pub
```

### Fix "REMOTE HOST IDENTIFICATION HAS CHANGED" in `known_hosts`

```bash
ssh-keygen -F app01.example.com
ssh-keygen -R app01.example.com
```

Removes only the rebuilt host's old entry instead of deleting the entire `known_hosts` file.

### Recover a missing `.pub` file or update a key's passphrase

```bash
ssh-keygen -y -f ~/.ssh/id_ed25519 > ~/.ssh/id_ed25519.pub
ssh-keygen -p -f ~/.ssh/id_ed25519
```

## Notes & Pitfalls

- Never copy or paste a private key (`id_ed25519`) onto remote servers—only copy the `.pub` file.
- Use `-o` / modern OpenSSH format (default since OpenSSH 7.8) and increase KDF rounds with `-a 100` for extra brute-force resistance on passphrases.

## Related Commands

- `ssh` — connect using your keys
- `ssh-copy-id` — install public keys on remote servers safely
- `ssh-agent` / `ssh-add` — cache unlocked private keys in memory
- `chmod` — enforce `700` on `~/.ssh` and `600` on private keys

## Additional Resources

- `man ssh-keygen`
