# Chapter 9 — Loops, Conditionals and Handlers

## Loops Without Shooting Yourself in the Foot

Modern Ansible uses `loop:` (replacing legacy `with_items` / `with_dict`). Always pair loops with `loop_control.label` so console output stays readable instead of dumping 200 lines of nested dictionary JSON per iteration.

```yaml
- name: Configure standard L3 interfaces
  cisco.ios.ios_l3_interfaces:
    config:
      - name: "{{ item.name }}"
        ipv4:
          - address: "{{ item.ip }}"
    state: merged
  loop: "{{ routed_interfaces }}"
  loop_control:
    label: "{{ item.name }} -> {{ item.ip }}"
```

> [!TIP]
> Whenever a network resource module accepts a **list** in `config:` (such as `ios_vlans`, `ios_interfaces`, `ios_l3_interfaces`), pass the entire list in **one task call** rather than looping over individual items. A single module call opens one config session, pushes one diff, and runs 5–10× faster.

### Looping Over Dictionaries (`dict2items`)

```yaml
- name: Display VRF route distinguishers
  ansible.builtin.debug:
    msg: "VRF {{ item.key }} uses RD {{ item.value.rd }}"
  loop: "{{ vrfs | dict2items }}"
  loop_control:
    label: "{{ item.key }}"
```

## Conditionals (`when:`)

Use `when:` to guard platform-specific tasks, version checks, or feature flags. Expressions inside `when:` are already evaluated as Jinja2—never wrap them in `{{ }}`.

```yaml
- name: Apply IOS-XE 17.x specific telemetry config
  cisco.ios.ios_config:
    lines:
      - telemetry ietf subscription 101
      - encoding encode-kvgpb
  when:
    - ansible_network_os == 'cisco.ios.ios'
    - ansible_net_version is version('17.3', '>=')
    - enable_streaming_telemetry | default(false) | bool
```

Note that a YAML list under `when:` acts as a logical **AND**—far easier to read and diff in Git than a 140-character single line with `and`.

## Handlers: Saving Config Only When Needed

Handlers are special tasks triggered via `notify:` only when a task reports `changed`. They run **once** at the end of a play (or immediately when `ansible.builtin.meta: flush_handlers` is called), preventing redundant `write memory` calls after every task.

```yaml
---
- name: Configure base management services
  hosts: ios
  gather_facts: false

  tasks:
    - name: Configure syslog servers
      cisco.ios.ios_logging_global:
        config:
          hosts:
            - host: 192.0.2.50
        state: merged
      notify: Save IOS configuration

    - name: Configure NTP servers
      cisco.ios.ios_ntp_global:
        config:
          servers:
            - server: 192.0.2.123
        state: merged
      notify: Save IOS configuration

  handlers:
    - name: Save IOS configuration
      cisco.ios.ios_config:
        save_when: always
```

If neither task changed anything, `write memory` never runs. If both changed, `write memory` runs **once** after both complete.
