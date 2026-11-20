---
title: 'Chapter 15: Canonical Academy SysAdmin Architecture & Exam Roadmap'
---

# Chapter 15: Canonical Academy SysAdmin Architecture & Exam Roadmap

Canonical Academy uses a **modular, role-based qualification framework** aligned directly with Ubuntu Long Term Support (LTS) releases (including **Ubuntu 26.04 LTS "Resolute Raccoon"**). Rather than a single multi-hour monolithic exam, candidates earn individual **Badges** by passing hands-on, browser-based terminal exams. Passing all four modular exams unlocks the capstone qualification: **Canonical Qualified SysAdmin (Canonical Academy SysAdmin)**.

```text
┌────────────────────────────────────────────────────────────────────────┐
│           CANONICAL ACADEMY SYSADMIN QUALIFICATION ARCHITECTURE        │
│                                                                        │
│  ┌─────────────────────────┐          ┌─────────────────────────┐      │
│  │ Badge 1 (75 mins)       │          │ Badge 2 (90 mins)       │      │
│  │ Using Linux Terminal    │          │ Using Ubuntu Desktop    │      │
│  │ • FHS, regex, pipes     │          │ • APT/DEB822 vs Snap    │      │
│  │ • Partitions, fstab     │          │ • Layer 2, Netplan/NM   │      │
│  │ • Journald, logrotate   │          │ • PKI, AD/SSSD, ACLs    │      │
│  │ • Users, DAC, sudo, SSH │          │ • GRUB2, LVM, Licensing │      │
│  └────────────┬────────────┘          └────────────┬────────────┘      │
│               │                                    │                   │
│               └─────────────────┬──────────────────┘                   │
│                                 ▼                                      │
│               ┌────────────────────────────────────┐                   │
│               │   CANONICAL QUALIFIED SYSADMIN     │                   │
│               │    (Capstone Credential Unlock)    │                   │
│               └─────────────────▲──────────────────┘                   │
│                                 │                                      │
│               ┌─────────────────┴──────────────────┐                   │
│               │                                    │                   │
│  ┌────────────┴────────────┐          ┌────────────┴────────────┐      │
│  │ Badge 3 (90 mins)       │          │ Badge 4 (90–120 mins)   │      │
│  │ Using Ubuntu Server     │          │ Using DevOps Principles │      │
│  │ • Autoinstall/Cloud-Init│          │ • Docker, LXD, KVM/virsh│      │
│  │ • Process & systemd     │          │ • CPU/Mem/IO & tcpdump  │      │
│  │ • SSH, UFW & iptables   │          │ • Bash, Python, Git     │      │
│  │ • Timers, NFS/SMB, Web  │          │ • Ansible & Ubuntu Pro  │      │
│  └─────────────────────────┘          └─────────────────────────┘      │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Exam Delivery & Scoring Mechanics

- **Format:** Mixed assessment combining **Performance-Based Labs** (live Ubuntu terminal tasks—highest point weight, partial credit awarded for every verified configuration step), **Scenario-Based Questions** (inspecting a live system state to answer diagnostic multiple-choice prompts—medium weight), and **Short-Answer/Multiple-Choice** (lowest weight).
- **Environment & Rules:** Browser-based proctored environment (AI + human-in-the-loop review). Local `man` pages, `--help` outputs, and built-in/official Ubuntu documentation (`/usr/share/doc/`) are accessible *inside* the exam environment; external browser tabs, proxies, or custom firewalls are prohibited.
- **Passing Standard:** Cut-score methodology calibrated by Subject Matter Experts (e.g., 70% cut score on baseline editions). Exams can be taken in any order.

| Exam / Badge | Duration | Status | Primary Operational Focus |
|---|---|---|---|
| **1. [Using Linux Terminal](16-cert-1-using-linux-terminal.md)** | 75 mins | General Availability | Core CLI mechanics, filesystem hierarchy, logs, storage partitioning, user/permission security. |
| **2. [Using Ubuntu Desktop](17-cert-2-using-ubuntu-desktop.md)** | 90 mins | General Availability | Package management (`apt` vs. `snap`), GRUB2/boot recovery, LVM, Layer 2 networking, AD/PKI, open-source governance. |
| **3. [Using Ubuntu Server](18-cert-3-using-ubuntu-server.md)** | 90 mins | General Availability | `systemd` service/timer orchestration, automated deployment (`PXE`, `autoinstall`, `cloud-init`), LAMP/LEMP stacks, NFS/Samba, `ufw`/`iptables`. |
| **4. [Using DevOps Principles](19-cert-4-using-devops-principles.md)** | 90–120 mins | Public Beta / GA Rollout | Bash/Python automation with Git, Ansible, Ubuntu Pro/ESM/Livepatch/Landscape, Docker/LXD/KVM (`virsh`), deep network/resource diagnostics. |

---

## Cross-Reference to Boring Linux Core Parts

| Canonical Academy Exam | Domain Modules & Weights | Corresponding Boring Linux Coverage |
|---|---|---|
| **1. Using Linux Terminal** | **1.1** Navigating Files & Filesystems (34%)<br>**1.2** Managing System Resources (33%)<br>**1.3** Securing Filesystem Access (33%) | **Part 01** (System Paths & State), **Part 04** (Users, Identity & Access), **Part 06** (Timers & Cron), **Part 36** (ext4 Internals), **Part 38 Ch 04–06** |
| **2. Using Ubuntu Desktop** | **2.1** Applications & Packages (28%)<br>**2.2** Networking Capabilities (25%)<br>**2.3** Securing Desktop Systems (22%)<br>**2.4** Install, Boot & LVM Recovery (15%)<br>**2.5** Open Source Community & Licensing (10%) | **Part 02** (Installation), **Part 03** (Boot, GRUB & Initramfs), **Part 04** (SSSD/Kerberos), **Part 05** (APT, DEB822, Snaps), **Part 38 Ch 01–07**, **Part 39** (Layer 2 ARP/VLANs) |
| **3. Using Ubuntu Server** | **3.1** Deploying Ubuntu Server (27%)<br>**3.2** Managing Processes & `systemd` (26%)<br>**3.3** Securing Server Access (24%)<br>**3.4** Configuring Servers & Services (23%) | **Part 02** (Unattended Install), **Part 03** (Rescue & Emergency Targets), **Part 06** (`systemd` Supervision & Timers), **Part 07** (Processes & Signals), **Part 38 Ch 05–07**, **Part 39** (Firewalls) |
| **4. Using DevOps Principles** | **4.1** Deployment Technologies (34%)<br>**4.2** Monitoring & Troubleshooting (34%)<br>**4.3** Automation & System Updating (32%) | **Part 05** (Livepatch & Security Updates), **Part 07** (Memory & PSI), **Part 20** (Virtualization), **Part 34** (Performance Engineering), **Part 38 Ch 08–14**, **Part 39** (Packet Forensics) |

---

## Recommended Study & 3-Node Lab Execution Roadmap

### 1. Local Lab Environment Setup (Multipass or LXD)

Deploy a local 3-node practice environment on Ubuntu 24.04 LTS / 26.04 LTS using **LXD** or **Multipass** so every destructive partitioning, GRUB rescue, firewall, and Ansible lab can be rehearsed safely:

```bash
# Option A: Provision 3 practice VMs with LXD (supports custom block volumes & nested KVM)
lxc launch ubuntu:26.04 desktop-node --vm -c limits.cpu=2 -c limits.memory=4GiB
lxc launch ubuntu:26.04 server-node  --vm -c limits.cpu=2 -c limits.memory=4GiB
lxc launch ubuntu:26.04 control-node --vm -c limits.cpu=4 -c limits.memory=8GiB

# Attach an extra 10GiB scratch block device to desktop-node and server-node for fdisk/gdisk/LVM drills
lxc storage volume create default scratch-desktop size=10GiB --type=block
lxc storage volume attach default scratch-desktop desktop-node
lxc storage volume create default scratch-server size=10GiB --type=block
lxc storage volume attach default scratch-server server-node
```

| Lab Instance | Target Certifications | Primary Practice Workloads |
|---|---|---|
| **`desktop-node`** | Cert 1 & Cert 2 | `fdisk`/`gdisk`/`parted`, `/etc/fstab`, GRUB2 & `initramfs` rescue, LVM snapshots (`pvcreate`/`vgcreate`/`lvcreate -s`), `nmcli`/Netplan, Snap confinement, POSIX ACLs, PKI trust store, and `realmd`/`sssd`. |
| **`server-node`** | Cert 1 & Cert 3 | `autoinstall` / `cloud-init`, single-user rescue (`systemd.unit=rescue.target`), custom `systemd` services & `.timer` backups, hardened `sshd_config`, `ufw`/`iptables`, NFSv4 exports, Samba shares, Nginx/Apache2, and PostgreSQL/MariaDB. |
| **`control-node`** | Cert 4 | Bash (`set -euo pipefail`) & Python automation, Git workflows, Ansible playbooks targeting `server-node` and `desktop-node`, Docker Compose stacks, LXD containers/VMs, `virsh` KVM management, `pro` CLI / Livepatch, and `vmstat`/`iostat`/`ss`/`dig`/`tcpdump` diagnostics. |

---

### 2. Exam Execution Strategy (Per Canonical Academy Guidelines)

1. **Triage First:** Scan all exam items at the start of the window and complete **Performance-Based Labs** first. They carry the highest point weight and award partial credit for every completed configuration artifact checked by the grading harness.
2. **Verify Persistence Across Reboots:** Grading scripts evaluate persistent configuration state, not just ephemeral runtime flags. Before finishing any lab task, run the four mandatory verification gates:
   ```bash
   # 1. Verify fstab mounts without rebooting
   sudo mount -a && findmnt --verify

   # 2. Verify systemd units are BOTH enabled at boot and currently active
   sudo systemctl daemon-reload
   sudo systemctl enable --now <service-or-timer>
   systemctl is-enabled <service-or-timer> && systemctl is-active <service-or-timer>

   # 3. Validate daemon configuration syntax before reloading
   sudo sshd -t
   sudo nginx -t
   sudo apache2ctl configtest
   sudo visudo -c
   testparm -s

   # 4. Verify firewall rules are active and persistent
   sudo ufw status numbered
   ```
3. **Use Built-In Terminal Documentation:** Never waste time guessing flag syntax. Use `man -k <keyword>`, `man 5 crontab`, `man 5 fstab`, `man 5 systemd.service`, `man 5 systemd.timer`, `<command> --help`, and example configs under `/usr/share/doc/` directly inside the exam terminal.
