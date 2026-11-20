---
title: 'Chapter 18: Certification 3 — Using Ubuntu Server'
---

# Chapter 18: Certification 3 — Using Ubuntu Server

**Target Profile:** Headless server deployment, `systemd` orchestration, network storage, web/database hosting, and perimeter hardening.  
**Duration:** 90 Minutes | **Format:** Performance Labs + Scenario & Knowledge Items

| Module | Topic Domain | Exam Weight |
|---|---|---|
| **Module 3.1** | Deploying Ubuntu Server | **27%** |
| **Module 3.2** | Managing Processes & `systemd` Units | **26%** |
| **Module 3.3** | Securing Server Access | **24%** |
| **Module 3.4** | Configuring Servers and Services | **23%** |

---

## Module 3.1: Deploying Ubuntu Server (Weight: 27%)

### 3.1.1 Automated & Unattended Provisioning

- **Ubuntu Autoinstall (Subiquity):** Authoring declarative YAML `autoinstall` configurations served via NoCloud (`user-data` + empty `meta-data` over HTTP or `cidata` volume) covering `version: 1`, `locale`, `keyboard`, `identity` (hostname, username, SHA-512 `password` hash from `mkpasswd -m sha-512`), `ssh` (`install-server: true`, `authorized-keys`), `storage` (`layout.name: lvm` or custom Curtin disk definitions), `packages`, and `late-commands` (running inside `curtin in-target --target=/target -- ...`).
- **Cloud-Init:** Authoring `#cloud-config` user-data for first-boot instance initialization (`users`, `package_update`, `package_upgrade`, `packages`, `write_files`, `runcmd`); inspecting boot stages and troubleshooting failures (`cloud-init status --wait`, `cloud-init analyze show`, `cloud-init query`, `/var/log/cloud-init.log`, `/var/log/cloud-init-output.log`, and resetting for image capture via `sudo cloud-init clean --logs`).
- **Network Boot (PXE):** Tracing the bare-metal PXE boot sequence: DHCP Discover/Offer (`next-server` TFTP IP + `filename` `grubx64.efi` / `pxelinux.0`) → TFTP download of UEFI bootloader, kernel (`vmlinuz`), and `initrd` → HTTP streaming of the Ubuntu Server Live ISO (`url=http://.../ubuntu-26.04-live-server-amd64.iso`) and `autoinstall` cloud-config seed (`ds=nocloud-net;s=http://.../seed/`).

```yaml
#cloud-config
# /var/www/html/seed/user-data (Ubuntu Server Autoinstall example)
autoinstall:
  version: 1
  locale: en_US.UTF-8
  keyboard:
    layout: us
  identity:
    hostname: ubuntu-srv-01
    username: ops
    password: "$6$rounds=4096$saltsalt$..."
  ssh:
    install-server: true
    allow-pw: false
    authorized-keys:
      - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... ops@control-node
  storage:
    layout:
      name: lvm
      sizing-policy: all
  packages:
    - ufw
    - nginx
    - vim
  late-commands:
    - curtin in-target --target=/target -- ufw allow OpenSSH
    - curtin in-target --target=/target -- systemctl enable nginx.service
```

### 3.1.2 Capacity Validation & Single-User Recovery

- **Pre-Deployment Hardware Validation:** Auditing CPU topology and virtualization flags (`lscpu`, `/proc/cpuinfo`, `kvm-ok`), physical RAM and swap allocation (`free -h`, `/proc/meminfo`, `vmstat`), block storage geometry and throughput (`lsblk -d -o NAME,ROTA,SIZE,MODEL`, `df -h`), and PCI/NIC link capabilities (`lspci`, `ethtool <iface>`).
- **Single-User / Rescue Booting:**
  1. Interrupt GRUB2 at boot (`Esc` or `Shift`), press `e` on the Ubuntu kernel entry, and append `systemd.unit=rescue.target` (or `systemd.unit=emergency.target` when `/etc/fstab` is broken, or `rw init=/bin/bash` if root is locked).
  2. Press `Ctrl-X` to boot into the root rescue shell.
  3. Remount the root filesystem read-write (`mount -o remount,rw /`), repair malformed `/etc/fstab` lines or reset a forgotten password (`passwd ops`), flush disk buffers (`sync`), and resume boot (`systemctl default` or `exec /sbin/init`).

---

## Module 3.2: Managing Processes & `systemd` Units (Weight: 26%)

### 3.2.1 Process Inspection & Signal Control

- **Process Telemetry:** Inspecting process states (`R` running, `S` interruptible sleep, `D` uninterruptible I/O sleep, `Z` zombie, `T` stopped), parent-child hierarchy, CPU/RSS memory usage, and open file descriptors/sockets (`ps aux --sort=-%mem`, `ps -ef`, `pstree -p`, `pgrep -a <name>`, `top`, `htop`, `lsof -p <PID>`, `lsof -i :80`).
- **Job & Signal Control:** Managing shell job control (`jobs -l`, `bg %1`, `fg %1`, `nohup <cmd> &`, `disown`), delivering POSIX signals (`kill -s TERM <PID>`, `pkill -HUP nginx`, `killall` using `SIGHUP` `1` reload, `SIGINT` `2` interrupt, `SIGTERM` `15` graceful termination, `SIGKILL` `9` untrappable kernel kill, `SIGSTOP`/`SIGCONT`), and adjusting CPU scheduler nice values (`-20` highest priority to `+19` lowest priority via `nice -n 10 <cmd>` and `sudo renice -n -5 -p <PID>`).

### 3.2.2 `systemd` Unit Lifecycle & Authoring

- **Unit States & Introspection:** Listing active, inactive, failed, and masked units (`systemctl list-units --type=service`, `systemctl list-unit-files`, `systemctl --failed`, `systemctl status <unit>`, `systemctl cat <unit>`, `systemctl list-dependencies <unit>`).
- **Service Control & Drop-In Overrides:** Starting, stopping, restarting, reloading, enabling, disabling, and masking units (`systemctl mask <unit>` symlinks to `/dev/null` so manual or dependency activation is impossible); creating upgrade-safe overrides in `/etc/systemd/system/<unit>.d/override.conf` via `sudo systemctl edit <unit>` (which overrides vendor files in `/usr/lib/systemd/system/`), and authoring standalone custom `.service` units followed by `sudo systemctl daemon-reload` and `sudo systemctl enable --now <unit>`.

```ini
# /etc/systemd/system/app-worker.service
[Unit]
Description=Internal Queue Worker Daemon
After=network-online.target postgresql.service
Wants=network-online.target

[Service]
Type=simple
User=appworker
Group=appworker
WorkingDirectory=/srv/app-worker
EnvironmentFile=-/etc/default/app-worker
ExecStart=/srv/app-worker/bin/worker --threads 4
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5s
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
ReadWritePaths=/var/lib/app-worker

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now app-worker.service
systemctl status app-worker.service --no-pager
```

---

## Module 3.3: Securing Server Access (Weight: 24%)

### 3.3.1 Hardening Remote SSH Access

- **Server Configuration:** Hardening `/etc/ssh/sshd_config` or modular drop-in files in `/etc/ssh/sshd_config.d/99-hardening.conf`:
  - `PermitRootLogin no`
  - `PasswordAuthentication no`
  - `PubkeyAuthentication yes`
  - `KbdInteractiveAuthentication no`
  - `AllowUsers ops deploy@10.0.0.0/24` (or `AllowGroups ssh-users`)
  - `MaxAuthTries 3`, `LoginGraceTime 30`, `ClientAliveInterval 300`, `ClientAliveCountMax 2`
- **Validation Gate:** Always run `sudo sshd -t` (and `sudo sshd -T | grep -E 'permitrootlogin|passwordauthentication|allowusers'`) before `sudo systemctl reload ssh.service`, while keeping an active SSH session open.

### 3.3.2 Regular Users vs. System Accounts

| Attribute | Regular Human User Account | System / Service Account |
|---|---|---|
| **Creation Command** | `useradd -m -s /bin/bash alice` / `adduser alice` | `useradd --system -s /usr/sbin/nologin -d /var/lib/svc svc` |
| **UID Range (`/etc/login.defs`)** | `UID >= 1000` (`1000`–`59999`) | `UID < 1000` (`100`–`999` dynamically allocated) |
| **Login Shell** | Interactive (`/bin/bash`, `/bin/zsh`) | Non-interactive (`/usr/sbin/nologin` or `/bin/false`) |
| **Password / Aging** | Hashed password or SSH key; aging policies apply | Locked password (`!` or `*` in `/etc/shadow`); no password expiry |
| **Purpose** | Interactive administration with least-privilege `sudo` | Isolate daemon process ownership so a service compromise cannot touch other services |

### 3.3.3 Host-Based Firewalls (`UFW` & `iptables`)

- **Uncomplicated Firewall (`ufw`):** Setting default policies (`sudo ufw default deny incoming`, `sudo ufw default allow outgoing`), allowing/limiting traffic by port, protocol, source CIDR, and interface (`sudo ufw limit 22/tcp`, `sudo ufw allow in on eth0 from 10.10.0.0/24 to any port 5432 proto tcp`, `sudo ufw allow "Nginx Full"`), inspecting numbered rules (`sudo ufw status numbered`), deleting rules (`sudo ufw delete <num>`), and enabling logging (`sudo ufw logging on`).
- **Netfilter / `iptables`:** Understanding tables (`filter`, `nat`) and chains (`INPUT`, `OUTPUT`, `FORWARD`, `PREROUTING`, `POSTROUTING`), rule evaluation order (first match wins, fallback to chain default policy), stateful connection tracking (`-m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT`), loopbacks (`-i lo -j ACCEPT`), targets (`ACCEPT`, `DROP` silent discard, `REJECT` immediate ICMP error), and persisting rules across reboots (`sudo iptables-save | sudo tee /etc/iptables/rules.v4` with `iptables-persistent` / `netfilter-persistent save`).

```bash
# UFW perimeter hardening for a web + restricted DB host
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw limit 22/tcp comment 'Rate-limit SSH'
sudo ufw allow 80,443/tcp comment 'HTTP/HTTPS Web'
sudo ufw allow from 10.10.20.0/24 to any port 5432 proto tcp comment 'App subnet to Postgres'
sudo ufw --force enable
sudo ufw status verbose

# Equivalent raw iptables INPUT chain logic
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -p tcp -m multiport --dports 80,443 -j ACCEPT
sudo iptables -A INPUT -s 10.10.20.0/24 -p tcp --dport 5432 -j ACCEPT
sudo iptables -P INPUT DROP
```

---

## Module 3.4: Configuring Servers and Services (Weight: 23%)

### 3.4.1 Automated Remote Backups (`systemd` Timers & `cron`)

- **Backup Tooling:** Writing idempotent `rsync` over SSH or compressed `tar` backup scripts (`rsync -aAXvz --delete -e "ssh -i /etc/backup/id_ed25519" /srv/data/ backup@10.10.30.50:/backups/srv-01/`).
- **`systemd` Timers:** Pairing a oneshot `.service` unit (`pg-backup.service`) with a `.timer` unit (`pg-backup.timer`) using `OnCalendar=*-*-* 02:30:00`, `Persistent=true` (catches up missed runs if the machine was powered off), and `RandomizedDelaySec=15m` (prevents fleet-wide backup stampedes); verifying with `systemd-analyze calendar "*-*-* 02:30:00"` and `systemctl list-timers --all`.

```ini
# /etc/systemd/system/srv-backup.service
[Unit]
Description=Nightly Rsync Backup of /srv/www

[Service]
Type=oneshot
User=root
ExecStart=/usr/bin/rsync -az --delete -e "ssh -i /root/.ssh/id_ed25519_backup" /srv/www/ backup@10.10.30.50:/backups/web-01/
```

```ini
# /etc/systemd/system/srv-backup.timer
[Unit]
Description=Run Nightly Rsync Backup at 02:15 UTC

[Timer]
OnCalendar=*-*-* 02:15:00
Persistent=true
RandomizedDelaySec=10m

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now srv-backup.timer
systemctl list-timers srv-backup.timer
```

### 3.4.2 Network File Sharing (NFS & Samba)

- **NFSv4 Server & Client:**
  - **Server (`nfs-kernel-server`):** Exporting directories in `/etc/exports` (`/srv/nfs/shared 10.10.20.0/24(rw,sync,no_subtree_check,root_squash)`), applying exports (`sudo exportfs -rav`), verifying active exports (`sudo exportfs -v`), and opening TCP port `2049` (`sudo ufw allow from 10.10.20.0/24 to any port nfs`).
  - **Client (`nfs-common`):** Mounting transiently (`sudo mount -t nfs4 10.10.20.10:/srv/nfs/shared /mnt/nfs`) and persistently in `/etc/fstab` (`10.10.20.10:/srv/nfs/shared /mnt/nfs nfs4 defaults,_netdev,nofail 0 0`).
- **Samba (`samba` SMB/CIFS):** Configuring `/etc/samba/smb.conf` share stanzas (`[teamshare]`, `path = /srv/samba/team`, `valid users = @sre-team`, `read only = no`, `browsable = yes`, `create mask = 0660`, `directory mask = 2770`), adding Samba users (`sudo smbpasswd -a devops1`, `sudo smbpasswd -e devops1`), validating syntax (`testparm -s`), restarting `smbd.service`, opening firewall (`sudo ufw allow Samba`), and testing client access (`smbclient -L //127.0.0.1 -U devops1`, `mount -t cifs -o credentials=/etc/samba/creds,uid=1000,gid=2500,_netdev`).

### 3.4.3 Web & Database Application Stacks

- **Web Servers (Nginx & Apache2):**
  - **Nginx:** Creating server blocks in `/etc/nginx/sites-available/app.conf`, symlinking into `/etc/nginx/sites-enabled/` (`sudo ln -s /etc/nginx/sites-available/app.conf /etc/nginx/sites-enabled/`), configuring TLS termination (`ssl_certificate`, `ssl_certificate_key`) and reverse proxying (`proxy_pass http://127.0.0.1:8080;`, `proxy_set_header Host $host;`, `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`), testing syntax (`sudo nginx -t`), and reloading (`sudo systemctl reload nginx`).
  - **Apache2:** Managing virtual hosts in `/etc/apache2/sites-available/app.conf`, enabling/disabling sites and modules (`sudo a2ensite app.conf`, `sudo a2dissite 000-default.conf`, `sudo a2enmod ssl proxy proxy_http rewrite headers`), testing syntax (`sudo apache2ctl configtest`), and reloading (`sudo systemctl reload apache2`).
- **Databases (PostgreSQL, MariaDB/MySQL):**
  - **PostgreSQL:** Binding `listen_addresses = 'localhost, 10.10.20.11'` in `/etc/postgresql/*/main/postgresql.conf`, configuring client CIDR + SCRAM authentication in `pg_hba.conf` (`host appdb appuser 10.10.20.0/24 scram-sha-256`), and creating roles/databases (`sudo -u postgres psql -c "CREATE ROLE appuser WITH LOGIN PASSWORD '...'; CREATE DATABASE appdb OWNER appuser;"`).
  - **MariaDB / MySQL:** Binding `bind-address = 127.0.0.1` in `/etc/mysql/mariadb.conf.d/50-server.cnf`, provisioning databases and least-privilege accounts (`CREATE DATABASE appdb; CREATE USER 'appuser'@'127.0.0.1' IDENTIFIED BY '...'; GRANT ALL PRIVILEGES ON appdb.* TO 'appuser'@'127.0.0.1'; FLUSH PRIVILEGES;`).

---

## Certification 3 Pre-Submission Verification Checklist

- [ ] **`systemd` Services & Timers:** Every custom `.service` and `.timer` unit is both `enabled` and `active` (`systemctl is-enabled <unit> && systemctl is-active <unit>`); timers show a valid next trigger timestamp in `systemctl list-timers`.
- [ ] **SSH & Firewall:** `sudo sshd -t` passes with zero output; `sudo ufw status` shows `Status: active` with port `22/tcp` explicitly allowed before closing your terminal.
- [ ] **NFS & Samba:** `/etc/exports` changes were applied with `sudo exportfs -rav`; `/etc/samba/smb.conf` passes `testparm -s`; client mounts in `/etc/fstab` include `_netdev` and pass `sudo mount -a`.
- [ ] **Web & Database:** `sudo nginx -t` or `sudo apache2ctl configtest` reports `Syntax OK`; `curl -I http://127.0.0.1` returns the expected HTTP status code; the application user can connect to the database over TCP (`psql -h 127.0.0.1 -U appuser -d appdb` or `mysql -h 127.0.0.1 -u appuser -p appdb`).
