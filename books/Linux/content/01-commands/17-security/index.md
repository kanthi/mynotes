---
title: Intro
---

# Intro

SELinux mode and file labels (Fedora/RHEL-family), AppArmor status (`aa-status` on Ubuntu/Debian/openSUSE), Linux file capabilities, Fail2ban (`fail2ban-client`), OpenPGP (`gpg`) for encrypt/sign/verify, modern file encryption with `age`, io_uring/seccomp policy, and unprivileged Landlock sandboxes from `setpriv`. Use `age` for operator file encryption (small keys, pipes, multi-recipient); use `gpg` when you need OpenPGP signatures or verifying signed checksum lists before trusting downloads. Use Landlock when a process should impose an extra filesystem (and, from ABI 4, TCP) allow-list on itself.

## Commands in this part

| Command | Role |
|---------|------|
| `getenforce` | Prints the current SELinux mode: Enforcing, Permissive, or Disabled. |
| `setenforce` | Switches SELinux between Enforcing (1) and Permissive (0) until reboot (or until changed again). |
| `restorecon` | Restores the default SELinux file contexts for paths based on policy file-context rules. |
| `getcap` / `setcap` | Linux file capabilities grant subsets of root privilege to executables. |
| `gpg` | OpenPGP encrypt, decrypt, sign, and verify — keys, armored files, and signed checksum workflows. |
| `age` | Simple modern file encryption — native or SSH recipients, armor, passphrase, tar pipes. |
| `setpriv --landlock-*` | Unprivileged Landlock sandbox: filesystem allow-lists from the command line; TCP via the Landlock syscalls (ABI 4+). |
| `aa-status` | Reports AppArmor status and loaded enforce/complain profiles on Debian, Ubuntu, and openSUSE. |
| `fail2ban-client` | Inspects Fail2ban jails and bans/unbans offending IPs for SSH and web services. |

## Suggested starting points

1. MAC Mode: `getenforce` / `setenforce` (SELinux) or `aa-status` (AppArmor).
2. Labels after copy/mv: `restorecon`.
3. Selective privilege on binaries: `getcap` / `setcap`.
4. Automated brute-force defense: `fail2ban-client status sshd`.
5. Download authenticity: `gpg --verify` on signed `SHA256SUMS`, then `sha256sum -c`.
6. Encrypt a backup tarball: `age-keygen`, then `tar … \| age -r …`.
7. Unprivileged filesystem sandbox: `setpriv --landlock-access fs` plus `path-beneath` rules (see the Landlock chapter).

## Related parts

- Users and groups — `sudo` and accounts
- Files and paths — DAC permissions, ACLs, and `sha256sum`
- Networking — host firewalls; `openssl` for TLS cert triage

Continue with the individual command pages in this part.
