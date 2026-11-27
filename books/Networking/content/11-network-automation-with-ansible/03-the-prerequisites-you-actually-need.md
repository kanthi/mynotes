# Chapter 3 — The Prerequisites You Actually Need

## Cutting Through the Noise

You do not need to be a senior software developer to automate a production network with Ansible. You *do* need fluency in five practical foundations: a Linux shell, SSH hygiene, YAML syntax, basic Git, and reading JSON/Python data structures.

## 1. Linux Command Line Essentials

Ansible's control node runs on Linux (or macOS / WSL2 on Windows). Master these daily operations:

```bash
# Navigate, inspect files, search recursively, and manage virtual environments
python3 -m venv .venv && source .venv/bin/activate
grep -rn "ntp server" group_vars/
find backups/ -name "*.cfg" -mtime +30
chmod 600 ~/.ssh/id_ed25519
```

Know how environment variables (`export ANSIBLE_CONFIG=./ansible.cfg`), exit codes (`echo $?`), pipes, and file permissions work—most early Ansible errors are path or permission problems.

## 2. SSH Fundamentals for Automation

Network automation lives on SSH. Understand how OpenSSH negotiates keys, ciphers, and jump hosts:

```ini
# ~/.ssh/config — clean Bastion/Jump host configuration for legacy and modern devices
Host 10.255.0.*
  User netops
  IdentityFile ~/.ssh/id_ed25519_netops
  ProxyJump bastion.corp.example
  ServerAliveInterval 30

Host legacy-ios-*
  KexAlgorithms +diffie-hellman-group14-sha1
  HostKeyAlgorithms +ssh-rsa
  PubkeyAcceptedAlgorithms +ssh-rsa
```

> [!WARNING]
> Older IOS/ASA images often require legacy key exchange (`diffie-hellman-group14-sha1`) or `ssh-rsa` host key algorithms that modern OpenSSH 9+ disables by default. Scope compatibility overrides strictly to legacy host patterns—never globally.

## 3. YAML Without the Pain

Ansible playbooks, inventories, and variable files are YAML. Remember five rules that eliminate 95% of syntax errors:

1. **Spaces only, never tabs.** Standard convention is 2 spaces per indentation level.
2. **Dictionaries (maps)** use `key: value` (note the required space after the colon).
3. **Lists (sequences)** use `- item` aligned at the same indentation.
4. **Quote strings that look like booleans, numbers, or contain colons/braces:** `"yes"`, `"010"`, `"10:20"`, `"{{ var }}"`.
5. **Multiline strings** use `|` (preserve newlines, ideal for banners/configs) or `>` (fold newlines into spaces).

```yaml
---
# Valid YAML combining mappings, lists, and a literal block scalar
device:
  hostname: sw-core-01
  mgmt_ip: 192.0.2.11
  vlans:
    - id: 10
      name: USERS
    - id: 20
      name: VOICE
  banner_motd: |
    Authorized access only.
    All sessions are logged.
```

## 4. Data Structures: Lists, Dictionaries, and JSON

Every structured fact returned by Ansible—whether from `ios_facts`, `junos_facts`, or NetBox—is a nested combination of **lists** (`[...]`) and **dictionaries** (`{...}`).

```json
{
  "ansible_net_hostname": "rtr-edge-01",
  "ansible_net_interfaces": {
    "GigabitEthernet1": {
      "ipv4": [{"address": "198.51.100.1", "subnet": "30"}],
      "operstatus": "up"
    }
  }
}
```

Accessing fields in Ansible/Jinja2 follows dot or bracket notation:
- `ansible_net_interfaces['GigabitEthernet1']['operstatus']` → `"up"`
- `ansible_net_interfaces['GigabitEthernet1'].ipv4[0].address` → `"198.51.100.1"`

## 5. Git Basics (The Safety Net)

Even as a solo engineer, keep every playbook and variable file in Git from day one:

```bash
git init
git checkout -b feat/standardize-ntp
git status
git diff
git add playbooks/ntp.yml group_vars/all.yml
git commit -m "Add standardized NTP playbook and site NTP servers"
```

## Hands-On Check: Verify Your Control Node Readiness

Run this quick check on your workstation or Linux jump host:

```bash
python3 --version
ssh -V
git --version
python3 -c "import yaml, json; print(yaml.safe_load('vlans: [10, 20, 30]'))"
```

If all four commands succeed cleanly, you have every prerequisite needed to build your lab and start automating.
