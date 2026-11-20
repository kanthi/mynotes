---
title: 'Chapter 16: Certification 1 — Using Linux Terminal'
---

# Chapter 16: Certification 1 — Using Linux Terminal

**Target Profile:** Operational command-line proficiency, resource maintenance, and local host security.  
**Duration:** 75 Minutes | **Format:** Performance Labs + Scenario & Knowledge Items

| Module | Topic Domain | Exam Weight |
|---|---|---|
| **Module 1.1** | Navigating Files and Filesystems | **34%** |
| **Module 1.2** | Managing System Resources | **33%** |
| **Module 1.3** | Securing Filesystem Access | **33%** |

---

## Module 1.1: Navigating Files and Filesystems (Weight: 34%)

### 1.1.1 Filesystem Hierarchy & Manipulation

- **FHS Mastery:** Navigating and understanding standard Linux directories:
  - `/etc` (host-specific static configuration), `/var` (variable state, logs, spools, caches), `/usr` (read-only userland binaries, libraries, documentation), `/opt` (add-on application packages), `/srv` (site-specific data served by the system), `/proc` (virtual kernel/process information filesystem), `/sys` (virtual `sysfs` device/kernel object hierarchy), `/run` (volatile runtime state in `tmpfs`), `/dev` (device nodes managed by `udev`), `/tmp` (volatile temporary files with sticky bit `1777`).
- **File Operations:** Creating, copying, moving, renaming, and deleting files and directory trees (`mkdir -p`, `cp -a`, `mv`, `rm -rf`, `touch`, `install -m 0755 -o root -g root`).
- **Linking:** Differentiating hard links vs. symbolic (soft) links (`ln`, `ln -s`), inode sharing behavior (`ls -i`, `stat`), cross-filesystem constraints (hard links cannot cross mount points or target directories), and resolving broken or nested symlinks (`readlink -f`, `find /path -xtype l`).
- **File Search & Metadata:** Locating binaries, manual pages, and files by metadata attributes (`which`, `whereis`, `type -a`, `locate`, `updatedb`, and `find` with `-name`, `-iname`, `-type`, `-size`, `-mtime`, `-mmin`, `-user`, `-perm`, `-exec ... {} +`).

```bash
# Create a directory hierarchy, preserve attributes during archive copy, and verify inodes
mkdir -p /srv/ops/{bin,conf,logs}
cp -a /etc/skel/. /srv/ops/conf/

# Hard link vs. symbolic link behavior
echo "production-config" > /srv/ops/conf/app.conf
ln /srv/ops/conf/app.conf /srv/ops/conf/app.conf.hard
ln -s /srv/ops/conf/app.conf /srv/ops/conf/app.conf.sym
stat -c '%i %n (%F)' /srv/ops/conf/app.conf*
readlink -f /srv/ops/conf/app.conf.sym

# Find all regular files under /var/log modified in the last 7 days larger than 10 MiB
find /var/log -type f -mtime -7 -size +10M -exec ls -lh {} +

# Find broken symlinks under /etc and print their targets
find /etc -xtype l -ls
```

### 1.1.2 Text Processing, Regular Expressions & Stream Redirection

- **Stream Redirection & Pipes:** Standard file descriptors (`stdin` `0`, `stdout` `1`, `stderr` `2`), redirection operators (`>`, `>>`, `2>`, `2>&1`, `&>`, `<`, `<<EOF` here-docs, `<<<` here-strings), `tee -a`, and `xargs -0`.
- **Regular Expressions (BRE & ERE):** Anchors (`^`, `$`, `\b`), character classes (`[a-z]`, `[^0-9]`, `[[:alpha:]]`, `[[:digit:]]`, `[[:space:]]`), quantifiers (`*`, `+`, `?`, `{n,m}`), grouping `(...)`, and alternation `|`.
- **CLI Text Utilities:** Filtering and transforming streams with `grep` (`-E`, `-i`, `-v`, `-r`, `-n`, `-o`, `-c`), `sed` (in-place substitution `sed -i 's/old/new/g'`, address ranges, line deletion `/pattern/d`), `awk` (`-F`, field extraction `$1, $NF`, `BEGIN`/`END` blocks, pattern-action filtering), `cut`, `sort` (`-n`, `-r`, `-k`, `-u`), `uniq -c`, `tr`, `wc`, `head`, `tail -f`, and `less`.
- **File Comparison & Archiving:** Comparing configurations and outputs (`diff -u`, `cmp`, `comm`, `md5sum`, `sha256sum -c`) and archiving/compression (`tar` with `-z` gzip, `-j` bzip2, `-J` xz, `--zstd` zstd).

```bash
# Redirect stdout and stderr separately, or tee combined output to a log file
grep -rn "Failed password" /var/log/ 2>/dev/null | tee /tmp/failed-ssh.log

# Extract top 5 offending source IPv4 addresses from auth.log using ERE, awk, sort, and uniq
grep -E "Failed password" /var/log/auth.log \
  | grep -Eo '([0-9]{1,3}\.){3}[0-9]{1,3}' \
  | sort | uniq -c | sort -nr | head -n 5

# In-place sed replacement with backup file creation and unified diff audit
cp /etc/ssh/sshd_config /tmp/sshd_config.lab
sed -i.bak -E 's/^#?PermitRootLogin.*/PermitRootLogin no/' /tmp/sshd_config.lab
diff -u /tmp/sshd_config.lab.bak /tmp/sshd_config.lab || true

# Create and verify a zstd-compressed tarball with sha256sum
tar --zstd -cf /tmp/ops-backup.tar.zst -C /srv ops
sha256sum /tmp/ops-backup.tar.zst > /tmp/ops-backup.tar.zst.sha256
sha256sum -c /tmp/ops-backup.tar.zst.sha256
```

---

## Module 1.2: Managing System Resources (Weight: 33%)

### 1.2.1 Storage Partitioning & Filesystem Maintenance

- **Block Device Inspection:** Identifying disks, partitions, filesystems, UUIDs, and active mount points (`lsblk -f`, `blkid`, `fdisk -l`, `df -hT`, `df -i`, `du -shx /*`, `findmnt`).
- **Partition Tables:** Creating and modifying MBR (`dos`) and GPT (`gpt`) partition layouts using `fdisk`, `gdisk`, and `parted` (setting partition types, ESP/LVM/swap flags, and non-interactive script mode `parted -s`).
- **Filesystem Creation & Repair:** Formatting volumes (`mkfs.ext4 -L data`, `mkfs.xfs -L fastdata`, `mkfs.vfat -F 32`, `mkswap`), tuning ext4 parameters (`tune2fs -l`, `-c`, `-m`), and checking/repairing **unmounted** filesystems (`fsck`, `e2fsck -f`, `xfs_repair`).
- **Persistent Mounting:** Configuring `/etc/fstab` using `UUID=` or `LABEL=`, security and lifecycle mount options (`defaults`, `noexec`, `nosuid`, `nodev`, `nofail`, `_netdev`), activating swap (`swapon -a`, `swapon --show`), and validating with `mount -a`, `findmnt --verify`, and `systemctl daemon-reload`.

```bash
# Non-interactive GPT partitioning on a scratch disk (/dev/sdb)
sudo parted -s /dev/sdb mklabel gpt
sudo parted -s /dev/sdb mkpart primary ext4 1MiB 4GiB
sudo parted -s /dev/sdb mkpart primary linux-swap 4GiB 6GiB
sudo partprobe /dev/sdb

# Format ext4 with label, reserve 1% blocks for root, and initialize swap
sudo mkfs.ext4 -L appdata /dev/sdb1
sudo tune2fs -m 1 /dev/sdb1
sudo mkswap -L swapvol /dev/sdb2

# Obtain UUID and add a hardened persistent mount to /etc/fstab
UUID_APPDATA=$(sudo blkid -s UUID -o value /dev/sdb1)
sudo mkdir -p /mnt/appdata
echo "UUID=${UUID_APPDATA} /mnt/appdata ext4 defaults,nosuid,nodev 0 2" | sudo tee -a /etc/fstab

# Always validate fstab before rebooting
sudo systemctl daemon-reload
sudo mount -a
findmnt --verify
```

### 1.2.2 System Logging & Log Rotation

- **Log Architecture:** Locating traditional rsyslog text logs in `/var/log/` (`syslog`, `auth.log`, `kern.log`, `dmesg`, `apt/history.log`, `apt/term.log`) vs. structured binary `systemd-journald` logs (`/run/log/journal` volatile vs. `/var/log/journal` persistent).
- **Journal Analysis:** Querying and filtering logs using `journalctl` (`-u <unit>`, `-p err..emerg`, `--since "1 hour ago"`, `--until`, `-b -1` previous boot, `-k` kernel messages, `-f` follow, `-o json-pretty`) and configuring persistent journal size caps in `/etc/systemd/journald.conf` (`Storage=persistent`, `SystemMaxUse=1G`).
- **Log Rotation:** Configuring `/etc/logrotate.conf` and service drop-in rules in `/etc/logrotate.d/` (directives: `daily`/`weekly`, `rotate`, `maxsize`, `compress`, `delaycompress`, `missingok`, `notifempty`, `create 0640 user group`, `sharedscripts`, `postrotate`/`endscript`) and testing with `logrotate -d` (dry-run) and `logrotate -vf` (forced execution).

```bash
# Filter journald for ssh.service errors since yesterday and check disk usage
journalctl -u ssh.service -p warning..emerg --since yesterday --no-pager
journalctl --disk-usage

# Create a custom logrotate drop-in for /var/log/custom-app/*.log
sudo tee /etc/logrotate.d/custom-app >/dev/null <<'EOF'
/var/log/custom-app/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        systemctl reload custom-app.service >/dev/null 2>&1 || true
    endscript
}
EOF

# Dry-run verify the logrotate policy
sudo logrotate -d /etc/logrotate.d/custom-app
```

### 1.2.3 Task Scheduling (`cron`)

- **Crontab Syntax & Structure:** Mastering the 5-field time specification (`minute hour day-of-month month day-of-week`), ranges (`1-5`), steps (`*/15`), lists (`1,15`), and special macros (`@reboot`, `@daily`, `@hourly`, `@weekly`).
- **System vs. User Crontabs:** Managing per-user crontabs (`crontab -e`, `crontab -l`, `crontab -u <user> -l`, `crontab -r`, stored in `/var/spool/cron/crontabs/`), system-wide `/etc/crontab` and `/etc/cron.d/*` (which include the **6th field: username** before the command), and `run-parts` periodic drop-in directories (`/etc/cron.{hourly,daily,weekly,monthly}`).
- **Access Control & Troubleshooting:** Restricting `crontab` usage via `/etc/cron.allow` (whitelist overrides deny) and `/etc/cron.deny`, setting explicit `PATH` and `SHELL` inside cron files, escaping literal `%` signs (`\%`) in cron command lines, and auditing execution via `journalctl -u cron.service` or `/var/log/syslog`.

```bash
# System drop-in in /etc/cron.d/ (requires explicit username field and escaped %)
sudo tee /etc/cron.d/db-health-snapshot >/dev/null <<'EOF'
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
*/15 * * * * root /usr/bin/df -h > /var/log/disk-snapshot-$(date +\%F).log 2>&1
EOF
sudo chmod 0644 /etc/cron.d/db-health-snapshot

# Restrict user crontabs strictly to alice and ops
printf "alice\nops\n" | sudo tee /etc/cron.allow >/dev/null
```

---

## Module 1.3: Securing Filesystem Access (Weight: 33%)

### 1.3.1 User, Group & Account Lifecycle Management

- **Account Databases:** Understanding the colon-delimited fields of `/etc/passwd` (`user:x:UID:GID:GECOS:home:shell`), `/etc/shadow` (password hash `$y$` yescrypt / `$6$` SHA-512, last change epoch days, min/max/warn/inactive/expire fields), `/etc/group`, and `/etc/gshadow`; querying via `getent passwd` / `getent group` and `id`.
- **CLI Account Administration:** Provisioning and modifying accounts with `useradd` (`-m`, `-s /bin/bash`, `-u`, `-g`, `-G`, `-c`, `-e`), `adduser`, `usermod` (`-aG` append supplementary group, `-L` lock, `-U` unlock, `-s`, `-d -m`), `userdel -r`, `groupadd`, `groupmod`, `groupdel`, and `gpasswd` (`-a`, `-d`, `-M`).
- **Password Aging & Complexity:** Enforcing password expiration policies via `chage` (`-l` list, `-M 90` max days, `-m 7` min days, `-W 14` warn days, `-I 7` inactive lock days, `-E YYYY-MM-DD` account expiry, `-d 0` force password change on next login), system defaults in `/etc/login.defs`, and PAM password complexity (`pam_pwquality` in `/etc/security/pwquality.conf`: `minlen`, `dcredit`, `ucredit`, `lcredit`, `ocredit`).

```bash
# Create group, provision user with supplementary group, and enforce 90-day password aging
sudo groupadd -g 2500 sre-team
sudo useradd -m -s /bin/bash -c "SRE Operator" -G sre-team,sudo devops1
sudo chage -m 7 -M 90 -W 14 -I 7 -E 2027-12-31 devops1
sudo chage -l devops1
```

### 1.3.2 Ownership, Permissions & Special Bits

- **Standard DAC Permissions:** Managing user/group ownership (`chown user:group`, `chown -R`, `chgrp`) and octal (`0750`, `0640`) or symbolic (`u=rwx,g=rx,o=`, `g+w,o-rwx`, `+X`) permissions (`chmod`).
- **Special Permission Bits:**
  - **SUID (`4000` / `u+s`)**: Executables run with the effective UID of the file owner (e.g., `/usr/bin/passwd`); audit across the filesystem with `find / -perm /4000 2>/dev/null`.
  - **SGID (`2000` / `g+s`)**: On shared team directories, newly created files and subdirectories automatically inherit the directory's group owner instead of the creator's primary group.
  - **Sticky Bit (`1000` / `+t`)**: On shared writable directories (like `/tmp` with `1777` or team dropboxes with `3770`), users can only delete or rename files they own.
- **Default Creation Masks:** Calculating and setting `umask` values (`0022` yields `0644` files / `0755` dirs; `0027` yields `0640` files / `0750` dirs) in `/etc/profile`, `/etc/bash.bashrc`, `/etc/login.defs`, and `~/.bashrc`.

```bash
# Create a shared collaboration directory with SGID (2000) + Sticky Bit (1000) = 3770
sudo mkdir -p /srv/sre-shared
sudo chown root:sre-team /srv/sre-shared
sudo chmod 3770 /srv/sre-shared
ls -ld /srv/sre-shared
# Output shows: drwxrws--T 2 root sre-team ... /srv/sre-shared
```

### 1.3.3 Privilege Escalation (`sudo`) & SSH Key Management

- **`sudo` Policy Engineering:** Safely editing `/etc/sudoers` and drop-in files in `/etc/sudoers.d/*` using `visudo -f /etc/sudoers.d/<name>`; interpreting rule syntax (`user MACHINE=(RUNAS_USER:RUNAS_GROUP) [NOPASSWD:] /full/path/to/command`), `Cmnd_Alias`, `env_reset`, `secure_path`, `sudo -l` privilege inspection, and auditing `sudo` invocations in `/var/log/auth.log` or `journalctl _COMM=sudo`.
- **SSH Key Authentication:** Generating Ed25519 or RSA keypairs (`ssh-keygen -t ed25519 -a 100 -C "user@host"`), deploying public keys (`ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host`), managing `~/.ssh/authorized_keys`, `~/.ssh/known_hosts` (`ssh-keygen -R`), `~/.ssh/config` host aliases, `ssh-agent` + `ssh-add`, and enforcing mandatory SSH filesystem permissions (`0700` on `~/.ssh`, `0600` on `~/.ssh/id_*` and `~/.ssh/authorized_keys`) so `sshd` `StrictModes` does not reject key login.

```bash
# Author a syntax-checked sudoers drop-in granting sre-team passwordless systemctl restart on nginx
sudo tee /etc/sudoers.d/sre-nginx >/dev/null <<'EOF'
%sre-team ALL=(root) NOPASSWD: /usr/bin/systemctl restart nginx.service, /usr/bin/systemctl status nginx.service
EOF
sudo chmod 0440 /etc/sudoers.d/sre-nginx
sudo visudo -cf /etc/sudoers.d/sre-nginx

# Enforce strict SSH directory and authorized_keys permissions for devops1
sudo install -d -m 0700 -o devops1 -g devops1 /home/devops1/.ssh
sudo touch /home/devops1/.ssh/authorized_keys
sudo chown devops1:devops1 /home/devops1/.ssh/authorized_keys
sudo chmod 0600 /home/devops1/.ssh/authorized_keys
```

---

## Certification 1 Pre-Submission Verification Checklist

- [ ] **Mounts & Partitions:** Every new volume in `/etc/fstab` uses `UUID=` or `LABEL=`, passes `sudo mount -a` with zero output, and reports clean in `findmnt --verify`.
- [ ] **Cron & Logrotate:** Custom files in `/etc/cron.d/` include the username column and have mode `0644`; logrotate rules pass `sudo logrotate -d /etc/logrotate.d/<file>`.
- [ ] **Users & Permissions:** Shared directories needing group inheritance have `chmod 2770` (or `3770` with sticky bit); `chage -l <user>` matches the prompt's aging policy.
- [ ] **Sudo & SSH:** Every `/etc/sudoers.d/*` file has mode `0440` and passes `sudo visudo -c`; `~/.ssh` is `0700` and `authorized_keys` is `0600`.
