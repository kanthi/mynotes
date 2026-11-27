# Chapter 33 — Secrets, Security and Governance

## Securing the Keys to the Kingdom

An automation platform that holds privileged credentials to every router, switch, and firewall in the enterprise is a **Tier-0 security asset**. Compromise of the control node or its secrets equals total network compromise.

## Managing Secrets: Ansible Vault vs. External Secrets Managers

| Approach | How It Works | Best For |
| :--- | :--- | :--- |
| **Ansible Vault** | AES-256 encrypted YAML files or inline encrypted strings committed to Git | Small teams, Git-only workflows, encrypted per-environment secret files |
| **AWX/AAP Credentials** | Encrypted in Postgres with AES; injected at runtime, never readable back via UI/API | Team execution with RBAC |
| **HashiCorp Vault / CyberArk** | Dynamic or centrally rotated secrets fetched via `community.hashi_vault` lookup or AWX external credential plugin | Enterprise governance, automatic password rotation, zero long-lived static secrets |

### Practical Ansible Vault Workflow

```bash
# Encrypt a single secret string to paste directly into group_vars/ios.yml
ansible-vault encrypt_string 'S3cr3tEnable!' --name 'vault_ios_enable_secret'

# Edit a fully encrypted vault file
ansible-vault edit inventory/group_vars/all/vault.yml

# Run a playbook using a vault password file stored outside the repo
ansible-playbook playbooks/site.yml --vault-password-file ~/.vault_pass
```

## Preventing Secret Leakage in Logs (`no_log: true`)

By default, Ansible prints task parameters and return values when `-v` or `--diff` is enabled. Always add `no_log: true` on any task handling SNMPv3 auth/priv passphrases, BGP MD5 keys, TACACS+ shared secrets, or local user hashes:

```yaml
- name: Configure TACACS+ server key without logging secret
  cisco.ios.ios_config:
    parents:
      - tacacs server ISE_PRIMARY
    lines:
      - "address ipv4 {{ tacacs_primary_ip }}"
      - "key 0 {{ vault_tacacs_shared_secret }}"
  no_log: true
```

## Governance Controls for Production Automation

1. **Dedicated automation identity:** Never run playbooks under a personal TACACS+ login. Use a service account (`svc-ansible-netops`) with per-command TACACS+ accounting and correlate logs with AWX job IDs and Git commits.
2. **Least-privilege separation:** Use read-only credentials for backup, fact-gathering, and compliance jobs; restrict write credentials strictly to approved deployment workflows.
3. **Hardened control plane:** Restrict SSH/HTTPS management ACLs on network devices so only the control node / AWX execution node IPs and break-glass bastion hosts can connect.
4. **Signed commits and branch protection:** Require GPG/SSH-signed commits and two-person approval on `main` for core routing and firewall repositories.
