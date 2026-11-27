# Chapter 7 — Your First Playbook

## Anatomy of a Playbook

A playbook is a YAML list of **plays**. Each play binds a group of inventory hosts to an ordered sequence of **tasks**. Every task invokes one module using its **Fully Qualified Collection Name (FQCN)**—for example `cisco.ios.ios_command` instead of the legacy short name `ios_command`.

```yaml
---
- name: Gather version and interface summary from IOS-XE devices
  hosts: ios
  gather_facts: false

  tasks:
    - name: Run show version and show ip interface brief
      cisco.ios.ios_command:
        commands:
          - show version | include Version
          - show ip interface brief
      register: cli_output

    - name: Display collected command output
      ansible.builtin.debug:
        msg:
          - "Host: {{ inventory_hostname }}"
          - "{{ cli_output.stdout_lines[0] }}"
```

## Running Your First Playbook

Execute the playbook with `ansible-playbook`:

```bash
# Syntax check without touching any device
ansible-playbook playbooks/first-check.yml --syntax-check

# Dry-run on a single host with verbose output
ansible-playbook playbooks/first-check.yml --limit rtr-edge-01 -v
```

### Understanding Task Statuses

| Status | Color | Meaning |
| :--- | :--- | :--- |
| **`ok`** | Green | Task ran successfully and no device state was modified |
| **`changed`** | Yellow | Task modified configuration or state on the target |
| **`skipped`** | Cyan | Task was bypassed because a `when:` condition evaluated false |
| **`failed`** | Red | Module returned an error; execution stops for that host unless handled |
| **`unreachable`** | Red | Control node could not establish SSH/NETCONF/HTTPS connectivity |

## Pushing Your First Idempotent Configuration Change

Once read-only commands succeed, move to a safe, idempotent configuration task—such as configuring NTP servers with a resource module:

```yaml
---
- name: Standardize NTP servers on Cisco IOS-XE
  hosts: ios
  gather_facts: false

  tasks:
    - name: Enforce corporate NTP servers
      cisco.ios.ios_ntp_global:
        config:
          servers:
            - server: 192.0.2.123
              prefer: true
            - server: 198.51.100.123
        state: replaced
      register: ntp_result

    - name: Save running configuration if NTP changed
      cisco.ios.ios_config:
        save_when: modified
      when: ntp_result.changed
```

Run it first with `--check --diff` to preview the exact IOS commands Ansible would send, then run it without `--check`, and run it a **second time** to confirm `changed=0` (proving idempotency).

> [!TIP]
> Always use `-v` (or `-vvv` for SSH/connection debugging) when developing new playbooks, and never skip `--check --diff` before applying changes to production groups.
