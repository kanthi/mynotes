# Chapter 10 — Roles and Reusability

## Why Monolithic Playbooks Fail

A 600-line playbook that configures NTP, SNMPv3, AAA, VLANs, and BGP in one file is impossible to test in isolation, painful to review in pull requests, and cannot be reused across sites. **Roles** break automation into self-contained, versionable building blocks with a predictable directory contract.

## Standard Role Directory Structure

Create a role skeleton with `ansible-galaxy role init roles/base_mgmt`:

```text
roles/base_mgmt/
├── defaults/
│   └── main.yml        # Lowest-precedence customizable defaults
├── handlers/
│   └── main.yml        # Save-config and reload handlers
├── meta/
│   └── main.yml        # Role metadata and dependencies
├── tasks/
│   ├── main.yml        # Entry point dispatching by ansible_network_os
│   ├── ios.yml         # Cisco IOS-XE implementation
│   ├── junos.yml       # Juniper Junos implementation
│   └── eos.yml         # Arista EOS implementation
├── templates/
│   ├── ios_base.j2     # Platform-specific Jinja2 templates
│   └── junos_base.j2
└── vars/
    └── main.yml        # Internal role constants
```

## Multi-Vendor Dispatch Pattern (`tasks/main.yml`)

The cleanest way to support multiple network operating systems in one role is dynamic task dispatch based on `ansible_network_os`:

```yaml
---
# roles/base_mgmt/tasks/main.yml
- name: Dispatch base management tasks per network OS
  ansible.builtin.include_tasks: "{{ platform_file }}"
  vars:
    platform_map:
      cisco.ios.ios: ios.yml
      junipernetworks.junos.junos: junos.yml
      arista.eos.eos: eos.yml
    platform_file: "{{ platform_map[ansible_network_os] }}"
  when: ansible_network_os in platform_map
```

## `import_role` vs. `include_role`

| Mechanism | Evaluation Time | Best Used For |
| :--- | :--- | :--- |
| **`import_role`** (static) | Parsed when the playbook loads | Standard static playbooks; `--list-tags` and `--list-tasks` see everything inside |
| **`include_role`** (dynamic) | Evaluated at runtime per host | Looping over roles or conditionally selecting roles based on discovered facts |

## Calling Roles from a Clean Top-Level Playbook

```yaml
---
- name: Enforce enterprise network baseline
  hosts: campus_switches
  gather_facts: false

  roles:
    - role: base_mgmt
      tags: [base, mgmt]
    - role: aaa_tacacs
      tags: [security, aaa]
    - role: campus_vlans
      tags: [vlans]
```

Now an engineer can run the entire baseline or target only VLAN updates with `--tags vlans`, while every team shares the same tested `base_mgmt` role across campus, WAN, and data centre inventories.
