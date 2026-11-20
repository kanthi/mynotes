---
title: 'Chapter 17: Certification 2 — Using Ubuntu Desktop'
---

# Chapter 17: Certification 2 — Using Ubuntu Desktop

**Target Profile:** Desktop/workstation administration, package ecosystems, boot recovery, networking, and enterprise directory integration.  
**Duration:** 90 Minutes | **Format:** Performance Labs + Scenario & Knowledge Items

| Module | Topic Domain | Exam Weight |
|---|---|---|
| **Module 2.1** | Managing Applications & Package Ecosystems | **28%** |
| **Module 2.2** | Configuring Networking Capabilities | **25%** |
| **Module 2.3** | Securing Desktop Systems | **22%** |
| **Module 2.4** | Installing Ubuntu Desktop & Boot/Storage Recovery | **15%** |
| **Module 2.5** | Engaging with the Open Source Community | **10%** |

---

## Module 2.1: Managing Applications & Package Ecosystems (Weight: 28%)

### 2.1.1 APT & `.deb` Package Management

- **Repository Configuration:** Managing legacy `/etc/apt/sources.list` and modern **DEB822** `.sources` files (`/etc/apt/sources.list.d/ubuntu.sources` with `Types: deb`, `URIs:`, `Suites:`, `Components: main restricted universe multiverse`, `Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg`), isolated third-party GPG keyrings in `/etc/apt/keyrings/`, and PPAs (`add-apt-repository ppa:user/repo`, `add-apt-repository --remove`).
- **Package Lifecycle:** Searching, inspecting, installing, holding, and purging packages using `apt` (`update`, `upgrade`, `full-upgrade`, `install`, `remove`, `purge`, `autoremove`, `search`, `show`, `list --installed`), `apt-cache` (`policy`, `depends`, `rdepends`), `apt-mark` (`hold`, `unhold`, `showhold`), and low-level `dpkg` (`-i <.deb>`, `-l`, `-L <pkg>` list installed files, `-S </path/file>` find owning package, `--configure -a` fix interrupted unpacks).
- **Automated Updates:** Configuring `unattended-upgrades` (`/etc/apt/apt.conf.d/50unattended-upgrades` and `/etc/apt/apt.conf.d/20auto-upgrades`) and testing with `sudo unattended-upgrade --dry-run --debug`.

```bash
# Inspect modern DEB822 repository configuration and package candidate priority
cat /etc/apt/sources.list.d/ubuntu.sources
apt-cache policy nginx

# Find which installed package owns a binary, hold a package version, and fix interrupted dpkg state
dpkg -S /usr/sbin/sshd
sudo apt-mark hold linux-image-generic
apt-mark showhold
sudo dpkg --configure -a
```

### 2.1.2 Snap Ecosystem vs. Traditional `.deb`

- **Architectural Contrast:**
  - **`.deb` (APT/dpkg):** Shared system libraries across FHS (`/usr/bin`, `/usr/lib`), single system-wide dependency graph resolved at install time, fine-grained maintainer scripts (`preinst`, `postinst`).
  - **`snap` (`snapd`):** Self-contained read-only SquashFS bundles mounted at `/snap/<name>/<rev>`, bundled runtime dependencies via base snaps (`core22`, `core24`, `core26`), transactional atomic updates with automatic rollback, mandatory AppArmor + seccomp + device cgroup confinement, and release channels (`track/risk/branch`, e.g., `latest/stable`, `latest/candidate`, `latest/beta`, `latest/edge`).
- **Snap Administration:** Searching, installing, refreshing, reverting, holding, and removing snaps (`snap find`, `snap info`, `snap install <pkg> --channel=...`, `snap install <pkg> --classic`, `snap list --all`, `snap refresh`, `snap refresh --hold=72h`, `snap revert <pkg>`, `snap disable`/`enable`, `snap remove --purge`).
- **Snap Interfaces & Permissions:** Inspecting confinement plugs (consumers) and slots (providers) and granting or revoking permissions (`snap connections <pkg>`, `snap interface`, `snap connect <snap>:<plug> <slot>`, `snap disconnect <snap>:<plug>`).

```bash
# Inspect snap channels, install from a specific channel, and audit confinement interfaces
snap info vlc
sudo snap install vlc --channel=latest/stable
snap connections vlc

# Connect an optional plug (e.g., removable-media) and hold automatic refreshes
sudo snap connect vlc:removable-media
sudo snap refresh --hold=48h vlc
```

---

## Module 2.2: Configuring Networking Capabilities (Weight: 25%)

### 2.2.1 Layer 2 Networking Principles

- **Frame & Link Layer Mechanics:** Ethernet frame structure, 48-bit MAC address anatomy (first 24 bits = IEEE **OUI** vendor identifier; last 24 bits = NIC serial; bit 0 of first octet = unicast `0` vs. multicast `1`; bit 1 = globally unique `0` vs. locally administered `1`), broadcast (`ff:ff:ff:ff:ff:ff`), and 802.1Q VLAN tagging.
- **Address Resolution Protocol (ARP) & NDP:** Inspecting and troubleshooting IPv4 ARP and IPv6 Neighbor Discovery tables (`ip neigh show`, `ip neigh flush dev <iface>`, `arping -I <iface> <ip>`) and diagnosing neighbor reachability states (`REACHABLE`, `STALE`, `DELAY`, `FAILED`, `INCOMPLETE`).

```bash
# Inspect link-layer MAC addresses, MTU, and neighbor (ARP/NDP) cache state
ip -br link show
ip neigh show
sudo ip neigh flush dev eth0
```

### 2.2.2 Ubuntu Network Configuration (`Netplan` & `NetworkManager`)

- **NetworkManager CLI (`nmcli`):** Managing desktop wired, Wi-Fi, and VPN profiles (`nmcli general status`, `nmcli dev status`, `nmcli con show`, `nmcli con add type ethernet con-name static-lan ifname eth0 ipv4.method manual ipv4.addresses 192.168.10.25/24 ipv4.gateway 192.168.10.1 ipv4.dns "1.1.1.1 9.9.9.9"`, `nmcli con mod`, `nmcli con up static-lan`).
- **Netplan Architecture:** Authoring declarative YAML in `/etc/netplan/*.yaml` (mode `0600`), selecting `renderer: NetworkManager` (desktop default) or `renderer: networkd` (server default), safely testing changes with automatic rollback via `sudo netplan try --timeout 30`, applying via `sudo netplan apply`, inspecting active merged config (`sudo netplan get`, `netplan status`), and verifying split-DNS resolution via `systemd-resolved` (`resolvectl status`, `resolvectl query`).

```yaml
# /etc/netplan/01-workstation.yaml (permissions must be 0600)
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 192.168.10.25/24
      routes:
        - to: default
          via: 192.168.10.1
      nameservers:
        search: [corp.internal]
        addresses: [192.168.10.2, 1.1.1.1]
```

```bash
sudo chmod 0600 /etc/netplan/01-workstation.yaml
sudo netplan generate
sudo netplan try --timeout 30
resolvectl status
```

---

## Module 2.3: Securing Desktop Systems (Weight: 22%)

### 2.3.1 Public Key Infrastructure (PKI)

- **PKI Components & Trust Chains:** Root CAs (self-signed anchors), Intermediate CAs (issuing authorities), leaf/server certificates, Certificate Signing Requests (CSRs), private keys, X.509 encoding formats (`.pem` / `.crt` Base64 ASCII, `.der` binary, `.p12` / `.pfx` PKCS#12 bundled cert + private key), and revocation checking via **CRLs** and **OCSP** stapling.
- **Trust Store Management:** Installing internal enterprise Root/Intermediate CA certificates into Ubuntu's system-wide trust anchor store (`/usr/local/share/ca-certificates/<name>.crt` with `.crt` extension in PEM format, followed by `sudo update-ca-certificates`) and inspecting certificate chains, SANs, and expiry dates with `openssl x509 -in cert.crt -noout -text -dates -subject -issuer`.

```bash
# Inspect an enterprise CA certificate and add it to Ubuntu's system trust store
openssl x509 -in /tmp/corp-root-ca.pem -noout -subject -issuer -dates
sudo cp /tmp/corp-root-ca.pem /usr/local/share/ca-certificates/corp-root-ca.crt
sudo chmod 0644 /usr/local/share/ca-certificates/corp-root-ca.crt
sudo update-ca-certificates
```

### 2.3.2 Active Directory (AD) Realm Integration

- **Domain Join Architecture:** Integrating Ubuntu Desktop into Microsoft Active Directory using `realmd`, `sssd` (`sssd-ad`, `sssd-tools`), `adcli`, `samba-common-bin`, `krb5-user`, and `pam_mkhomedir` (`pam-auth-update --enable mkhomedir`).
- **Operational Commands:** Discovering AD realms over DNS SRV records (`realm discover corp.example.com`), joining domains (`sudo realm join -U Administrator corp.example.com`), restricting login access to specific AD groups (`sudo realm permit -g "Domain Admins" "Linux Workstations"`), tuning `/etc/sssd/sssd.conf` (`use_fully_qualified_names = False`, `fallback_homedir = /home/%u@%d`, mode `0600`), clearing SSSD cache (`sudo sss_cache -E`), and verifying Kerberos TGT acquisition (`kinit user@CORP.EXAMPLE.COM`, `klist`).

```bash
# Discover AD domain, join via realmd, enable auto-home creation, and permit an AD security group
realm discover corp.example.com
sudo realm join -U Administrator corp.example.com
sudo pam-auth-update --enable mkhomedir
sudo realm permit -g "linux-admins@corp.example.com"
id "alice@corp.example.com"
```

### 2.3.3 Access Control Lists (ACLs)

- **Extended Permissions:** Granting fine-grained per-user and per-group permissions beyond a single owning UID/GID using POSIX ACLs (`getfacl`, `setfacl -m u:alice:rwx`, `setfacl -m g:auditors:r-x`, `setfacl -x` remove entry, `setfacl -b` strip all extended ACLs).
- **Default ACLs & Masks:** Configuring inheritable **default ACLs** (`setfacl -m d:u:alice:rwx,d:g:auditors:r-x /srv/project`) on directories so newly created files/subdirectories inherit the policy, and understanding the **ACL mask** (`mask::r-x`), which acts as the upper ceiling on effective permissions for named users, the owning group, and named groups.

```bash
# Grant auditor read-only access and set inheritable default ACLs on /srv/finance
sudo mkdir -p /srv/finance
sudo setfacl -R -m g:auditors:r-x /srv/finance
sudo setfacl -R -m d:g:auditors:r-x,d:u:alice:rwx /srv/finance
getfacl /srv/finance
```

---

## Module 2.4: Installing Ubuntu Desktop & Boot/Storage Recovery (Weight: 15%)

### 2.4.1 Installation, Media & Hardware Drivers

- **Installation & Release Upgrades:** Verifying ISO SHA-256 checksums and GPG signatures, writing bootable USB media (`sudo dd if=ubuntu-26.04-desktop-amd64.iso of=/dev/sdX bs=4M status=progress oflag=sync`), and performing LTS-to-LTS upgrades (`sudo do-release-upgrade`, `/etc/update-manager/release-upgrades` with `Prompt=lts`).
- **Proprietary Drivers:** Detecting PCI/USB devices (`lspci -nnk`, `lsusb`) and installing proprietary NVIDIA GPU or Broadcom Wi-Fi drivers using `ubuntu-drivers devices`, `sudo ubuntu-drivers install`, and inspecting DKMS kernel module rebuilds (`dkms status`).

### 2.4.2 Bootloader (GRUB2) & Initramfs Troubleshooting

- **GRUB2 Configuration:** Modifying `/etc/default/grub` (`GRUB_TIMEOUT`, `GRUB_CMDLINE_LINUX_DEFAULT`, `GRUB_DISABLE_RECOVERY`), compiling `/boot/grub/grub.cfg` via `sudo update-grub` (`grub-mkconfig -o /boot/grub/grub.cfg`), and reinstalling the bootloader binary with `sudo grub-install /dev/sda` (or `--efi-directory=/boot/efi`).
- **Rescuing from `grub>` / `grub rescue>` & Initramfs Rebuild:** Locating the root partition manually at the GRUB prompt (`ls`, `set root=(hd0,gpt2)`, `linux /boot/vmlinuz-... root=/dev/sda2`, `initrd /boot/initrd.img-...`, `boot`), or mounting a broken system via Live USB chroot (`mount /dev/sda2 /mnt`, bind-mounting `/dev`, `/proc`, `/sys`, `/run`, `chroot /mnt`) and regenerating initramfs images with `sudo update-initramfs -u -k all`.

### 2.4.3 Logical Volume Management (LVM)

- **LVM Stack Lifecycle:** Initializing Physical Volumes (`pvcreate /dev/sdb1`, `pvs`, `pvdisplay`), creating and extending Volume Groups (`vgcreate vg_data /dev/sdb1`, `vgextend vg_data /dev/sdc1`, `vgs`), creating Logical Volumes (`lvcreate -L 5G -n lv_docs vg_data`, `lvs`), and online volume + filesystem expansion (`sudo lvextend -r -L +2G /dev/vg_data/lv_docs` which automatically invokes `resize2fs` or `xfs_growfs`).
- **LVM Copy-on-Write Snapshots:** Creating point-in-time snapshots before risky changes (`sudo lvcreate -s -L 1G -n lv_docs_snap /dev/vg_data/lv_docs`), mounting snapshots read-only for file recovery, or rolling back the origin volume (`sudo umount /mnt/docs && sudo lvconvert --merge /dev/vg_data/lv_docs_snap`).

```bash
# Build an LVM stack, format ext4, grow online by 2 GiB, and take a CoW snapshot
sudo pvcreate /dev/sdc1
sudo vgcreate vg_workstation /dev/sdc1
sudo lvcreate -L 4G -n lv_home_projects vg_workstation
sudo mkfs.ext4 /dev/vg_workstation/lv_home_projects
sudo mkdir -p /mnt/projects
sudo mount /dev/vg_workstation/lv_home_projects /mnt/projects

# Online expansion of both Logical Volume and ext4 filesystem in one command (-r)
sudo lvextend -r -L +2G /dev/vg_workstation/lv_home_projects

# Create a 512 MiB snapshot before a risky migration
sudo lvcreate -s -L 512M -n lv_projects_pre_mig /dev/vg_workstation/lv_home_projects
sudo lvs
```

---

## Module 2.5: Engaging with the Open Source Community (Weight: 10%)

### 2.5.1 Ubuntu Release Cycle & Governance

- **Versioning & Support Windows:**
  - **Ubuntu `YY.MM` format:** Even-year April releases (`22.04 LTS`, `24.04 LTS`, `26.04 LTS`) are **Long Term Support (LTS)** releases receiving **5 years** of standard security maintenance in `Main`, extendable to **10 years** via **Ubuntu Pro (Expanded Security Maintenance / ESM)** plus an additional 2-year Legacy add-on (12 years total).
  - **Interim Releases:** Published every October and odd-year April (e.g., `25.04`, `25.10`), supported for **9 months** to introduce newer kernels, toolchains, and desktop stacks.
- **Governance & Bug Reporting:** Canonical + Ubuntu Community Council + Ubuntu Technical Board; filing diagnostic telemetry and crash reports to Launchpad using `ubuntu-bug <package>` (`apport-cli`).

### 2.5.2 Open Source Software Licensing

| License Category | Representative Licenses | Derivative Work & Redistribution Obligations |
|---|---|---|
| **Strong Copyleft** | **GPLv2**, **GPLv3**, **AGPLv3** | Any distributed derivative work or linked binary must be licensed under the same GPL license and make complete corresponding source code available. **AGPLv3** closes the SaaS/network loophole by triggering source disclosure when users interact with the modified software over a network. |
| **Weak / File-Level Copyleft** | **LGPLv2.1/v3**, **MPL 2.0** | Modifications to the covered library/files themselves must remain open under the same license, but proprietary applications may dynamically link (**LGPL**) or combine separate files (**MPL 2.0**) without open-sourcing the proprietary work. |
| **Permissive** | **MIT**, **BSD 2/3-Clause**, **Apache 2.0** | Allows redistribution in open or proprietary form with minimal restriction, provided copyright and license notices are preserved. **Apache 2.0** additionally provides an explicit **patent grant** and patent retaliation clause, and requires documenting modifications. |

---

## Certification 2 Pre-Submission Verification Checklist

- [ ] **APT & Snap:** `sudo apt update` completes with zero GPG or DEB822 syntax errors; held packages show in `apt-mark showhold`; required snap plugs show connected in `snap connections <snap>`.
- [ ] **Netplan / NetworkManager:** `/etc/netplan/*.yaml` has permissions `0600`; `sudo netplan get` and `resolvectl status` reflect the exact static IP, gateway, and DNS servers requested.
- [ ] **PKI, AD & ACLs:** Custom CAs are copied as `.crt` into `/usr/local/share/ca-certificates/` and hashed via `sudo update-ca-certificates`; `getfacl <dir>` shows both active and `default:` ACL entries with an unrestrictive `mask::`.
- [ ] **GRUB2 & LVM:** Any edit to `/etc/default/grub` was compiled with `sudo update-grub`; resized LVM volumes show the new filesystem capacity in **both** `lvs` and `df -h` (confirming `lvextend -r` or `resize2fs`/`xfs_growfs` was run).
