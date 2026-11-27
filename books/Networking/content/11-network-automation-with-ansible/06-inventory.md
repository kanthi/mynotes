# Chapter 6 — Inventory

## What an Inventory Actually Is

An Ansible inventory is more than a list of IP addresses—it is the **structural graph of your network**. It organizes devices into overlapping groups by platform, role, site, and environment, and attaches connection parameters and variables at each level.

## YAML Inventory Structure

Prefer YAML (`inventory/hosts.yml`) over legacy INI format. YAML supports nested dictionaries, lists, and typed booleans natively without quoting gymnastics.

```yaml
---
all:
  children:
    # Group by vendor / network OS
    ios:
      vars:
        ansible_network_os: cisco.ios.ios
        ansible_connection: ansible.netcommon.network_cli
        ansible_become: true
        ansible_become_method: enable
      hosts:
        sw-acc-01:
          ansible_host: 172.20.20.11
        sw-acc-02:
          ansible_host: 172.20.20.12
        rtr-edge-01:
          ansible_host: 172.20.20.21

    junos:
      vars:
        ansible_network_os: junipernetworks.junos.junos
        ansible_connection: ansible.netcommon.netconf
        ansible_port: 830
      hosts:
        rtr-wan-01:
          ansible_host: 172.20.20.31

    # Group by functional role (same hosts referenced across multiple groups)
    access_switches:
      hosts:
        sw-acc-01: {}
        sw-acc-02: {}

    edge_routers:
      hosts:
        rtr-edge-01: {}
        rtr-wan-01: {}

    # Group by physical site
    site_london:
      children:
        access_switches: {}
        edge_routers: {}
```

## Separating Data from Structure: `group_vars/` and `host_vars/`

Keep `inventory/hosts.yml` lean—only hostnames, `ansible_host` IPs, and group membership. Move all configuration data into sibling `group_vars/` and `host_vars/` directories:

```text
inventory/
├── hosts.yml
├── group_vars/
│   ├── all.yml
│   ├── ios.yml
│   ├── junos.yml
│   ├── access_switches.yml
│   └── site_london.yml
└── host_vars/
    ├── rtr-edge-01.yml
    └── sw-acc-01.yml
```

## Targeting Hosts with Patterns

Ansible host patterns let you slice your inventory with set logic—both inside a playbook's `hosts:` field and via `--limit` on the command line:

| Pattern | Meaning |
| :--- | :--- |
| `all` | Every host in the inventory |
| `site_london` | All hosts in `site_london` and its child groups |
| `site_london:&ios` | **Intersection:** hosts in *both* `site_london` AND `ios` |
| `site_london:!edge_routers` | **Exclusion:** hosts in `site_london` EXCEPT `edge_routers` |
| `sw-acc-*` | Wildcard match on inventory hostname |
| `access_switches[0]` | First host in `access_switches` (great for canary testing) |

## Inspecting Your Inventory with `ansible-inventory`

Before running a playbook, always verify how Ansible actually resolved group hierarchy and variables:

```bash
# Visual group tree
ansible-inventory -i inventory/hosts.yml --graph

# Inspect every merged variable for a single host
ansible-inventory -i inventory/hosts.yml --host sw-acc-01
```

## Dynamic Inventories

Static YAML inventories work well up to a few dozen devices, after which they inevitably drift from reality. Dynamic inventory plugins pull live device lists, IPs, roles, and tags directly from a Source of Truth like **NetBox** (`netbox.netbox.nb_inventory`), **Nautobot**, or cloud providers. We build a complete NetBox dynamic inventory in [Chapter 19 — Source of Truth](19-source-of-truth.md).
