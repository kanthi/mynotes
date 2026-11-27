# Chapter 15 — Resource Modules in Depth

## Why Resource Modules Beat Raw CLI Configs

Raw `ios_config` compares text lines—it cannot easily detect that an old IP address should be removed, or normalize `Gi0/1` vs. `GigabitEthernet0/1`. **Network Resource Modules** solve this by separating a device into discrete, schema-validated resources (interfaces, VLANs, LACP, BGP, ACLs, NTP, static routes) and converting live CLI into structured Python/YAML facts before computing the minimal delta.

## The Seven Resource Module States

Mastering the `state:` parameter is critical—picking the wrong state can wipe untouched interfaces or VLANs.

| State | Behavior | Production Risk |
| :--- | :--- | :--- |
| **`merged`** | Adds or updates declared attributes; leaves everything else untouched | Low (default, additive) |
| **`replaced`** | Replaces the entire config of *only the listed resource instances* (e.g., `GigabitEthernet1/0/1`); leaves unlisted interfaces alone | Medium |
| **`overridden`** | Makes the **entire** resource type on the device match `config:`—deletes any unlisted VLANs, interfaces, or ACLs! | **High** (use with extreme care) |
| **`deleted`** | Removes listed resource instances (or *all* instances if `config:` is omitted) | **High** |
| **`gathered`** | Reads live device config and returns it in the exact resource module YAML schema | None (read-only) |
| **`rendered`** | Converts `config:` YAML into vendor CLI commands offline without touching any device | None (offline) |
| **`parsed`** | Parses a raw running-config text string offline into structured resource YAML | None (offline) |

> [!CAUTION]
> Calling `ios_l3_interfaces` with `state: overridden` and only listing `GigabitEthernet0/1` will strip the IP address off your management interface too—locking you out immediately. Use `state: replaced` for scoped resources, and reserve `overridden` for strictly modeled resources like `ios_vlans` or `ios_ntp_global` where your Source of Truth owns 100% of entries.

## Brownfield Onboarding with `state: gathered`

Starting from an existing production network? Use `state: gathered` to export live device configurations directly into ready-to-use `host_vars/` YAML:

```yaml
- name: Gather existing VLANs in resource module format
  cisco.ios.ios_vlans:
    state: gathered
  register: gathered_vlans

- name: Save gathered VLANs to host_vars
  ansible.builtin.copy:
    content: "{{ {'vlans': gathered_vlans.gathered} | to_nice_yaml(indent=2) }}"
    dest: "inventory/host_vars/{{ inventory_hostname }}_vlans.yml"
    mode: "0644"
  delegate_to: localhost
```

## Example: Managing Interfaces and VLANs Declaratively

```yaml
- name: Enforce access switch VLAN database
  cisco.ios.ios_vlans:
    config:
      - vlan_id: 10
        name: CORP_USERS
        state: active
      - vlan_id: 20
        name: CORP_VOICE
        state: active
      - vlan_id: 99
        name: MGMT
        state: active
    state: merged

- name: Enforce L2 switchport configuration on user ports
  cisco.ios.ios_l2_interfaces:
    config:
      - name: GigabitEthernet1/0/1
        mode: access
        access:
          vlan: 10
        voice:
          vlan: 20
    state: replaced
```

Always inspect `result.commands`, `result.before`, and `result.after` in `--check --diff` mode before promoting a resource module change to production.
