# Chapter 13 — Authentication and Privilege

## Credentials Without Hardcoding

Never commit plaintext passwords or enable secrets into `inventory/hosts.yml` or `group_vars/`. Pass credentials via **Ansible Vault**, environment variables, or an external secrets manager (HashiCorp Vault, AWX Credentials, CyberArk).

```yaml
# group_vars/ios.yml (references vaulted variables stored in group_vars/ios/vault.yml)
ansible_user: "{{ vault_netops_user }}"
ansible_password: "{{ vault_netops_password }}"
ansible_become: true
ansible_become_method: enable
ansible_become_password: "{{ vault_ios_enable_secret }}"
```

## Privilege Escalation (`enable` vs. Direct Priv 15)

Cisco IOS/IOS-XE and ASA distinguish between user EXEC (`>`) and privileged EXEC (`#`, privilege level 15):

- **Option A — Direct privilege 15 via TACACS+/RADIUS:** Preferred for automation accounts. The user lands straight at `#`, eliminating `enable` prompts and simplifying `network_cli`.
- **Option B — `ansible_become: true` with `ansible_become_method: enable`:** Required when AAA policy drops logins into privilege level 1.

> [!WARNING]
> If your TACACS+ server enforces **command authorization**, verify that the automation service account is permitted to run `terminal length 0`, `terminal width 512`, `show running-config`, and `configure terminal`—otherwise `network_cli` fails during initial session setup before your first task even runs.

## SSH Key Authentication on Network Devices

Where supported (IOS-XE 16/17+, NX-OS, Junos, Arista EOS, FortiOS), prefer SSH public-key authentication over static passwords:

```yaml
# group_vars/junos.yml
ansible_user: netops-svc
ansible_ssh_private_key_file: ~/.ssh/id_ed25519_netops
```

On Cisco IOS-XE, note that older releases only accept RSA keys (2048–4096 bit) formatted in RFC 4716 or OpenSSH PEM; verify key support per OS release before removing password fallback.

## Token-Based Authentication for REST/HTTPAPI

API-first devices like **FortiGate (FortiOS)** and **Cisco Meraki** authenticate with scoped bearer tokens instead of interactive SSH passwords:

```yaml
# group_vars/fortios.yml
ansible_connection: ansible.netcommon.httpapi
ansible_network_os: fortinet.fortios.fortios
ansible_httpapi_use_ssl: true
ansible_httpapi_validate_certs: true
ansible_httpapi_port: 443
ansible_fortios_access_token: "{{ vault_fortios_api_token }}"
```

Always bind REST API tokens to a dedicated **Trusted Hosts** subnet (your control node / AWX execution node IPs) and a least-privilege admin profile.
