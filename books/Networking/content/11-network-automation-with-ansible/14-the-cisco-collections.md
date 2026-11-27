# Chapter 14 — The Cisco Collections

## Choosing the Right Cisco Collection

Cisco maintains separate, platform-specific collections on Ansible Galaxy. Never mix modules from one OS against another—their CLI grammars, datastore models, and commit semantics differ completely.

| Platform | Collection FQCN | Default Connection | Config Model |
| :--- | :--- | :--- | :--- |
| **IOS / IOS-XE** (Catalyst, ISR, CSR, Cat8000v) | `cisco.ios` | `network_cli` | Immediate apply to running-config; explicit `write memory` |
| **NX-OS** (Nexus 9000/3000) | `cisco.nxos` | `network_cli` or `httpapi` (NX-API) | Immediate apply + `copy running-config startup-config`; supports `checkpoint` rollback |
| **IOS-XR** (ASR 9000, NCS, 8000) | `cisco.iosxr` | `network_cli` or `netconf` | Candidate config + explicit `commit` (supports `commit confirmed`) |
| **ASA** (ASA / ASAv) | `cisco.asa` | `network_cli` | Immediate apply; object-groups and ACLs require careful ordering |

## The Three Module Families in `cisco.ios`

1. **Command modules (`ios_command`):** Execute operational `show` commands and register stdout for assertions or parsing.
2. **Config modules (`ios_config`):** Push raw CLI lines or Jinja2 templates with `parents:` context, `before:`/`after:` hooks, and `match:`/`replace:` logic. Ideal for features without a dedicated resource module.
3. **Resource modules (`ios_vlans`, `ios_interfaces`, `ios_l2_interfaces`, `ios_l3_interfaces`, `ios_bgp_global`, `ios_acls`, `ios_ospfv2`):** Parse device state into structured YAML and compute idempotent diffs automatically.

## Using `ios_config` Safely with `parents` and `diff_against`

When a feature lacks a resource module, `cisco.ios.ios_config` remains your workhorse. Always specify `parents:` for sub-mode configuration so Ansible compares inside the right hierarchy:

```yaml
- name: Configure VTY line access-class and SSH transport
  cisco.ios.ios_config:
    parents:
      - line vty 0 15
    lines:
      - access-class MGMT_SSH_IN in vrf-also
      - transport input ssh
      - exec-timeout 15 0
    match: exact
    diff_against: running
    save_when: changed
```

## NX-OS Checkpoints and Rollbacks

On Cisco Nexus (`cisco.nxos`), leverage native checkpoints before risky changes:

```yaml
- name: Create NX-OS rollback checkpoint
  cisco.nxos.nxos_command:
    commands:
      - no checkpoint file bootflash:pre_change.cfg
      - checkpoint file bootflash:pre_change.cfg
  ignore_errors: false
```

If post-change verification fails in a `rescue:` block, `rollback running-config file bootflash:pre_change.cfg atomic` restores the exact pre-change state in seconds.
