# Chapter 20 — Template-Driven Config Generation

## Separating Config Generation from Config Deployment

Never render a Jinja2 template directly onto a live router in a single blind step. Split the pipeline into two distinct stages:

1. **Build stage (offline, zero risk):** Render every device's full or modular configuration into `intended/configs/<hostname>.cfg` on the control node. Inspect diffs in Git, run static analyzers (Batfish), or deploy into Containerlab first.
2. **Deploy stage (controlled, atomic):** Push the reviewed, pre-rendered configuration file to the device using `replace` or `merge` semantics with `--check --diff`.

## Modular Master Template Pattern

Break large device configs into clean, composable partials included by a single platform master template (`templates/ios/main.j2`):

```jinja2
! ==========================================================
! Managed by Ansible — DO NOT EDIT MANUALLY
! Hostname: {{ inventory_hostname }} | Site: {{ site_code }}
! ==========================================================
hostname {{ inventory_hostname }}
!
{% include 'ios/base_mgmt.j2' %}
{% include 'ios/vlans.j2' %}
{% include 'ios/interfaces.j2' %}
{% include 'ios/routing_bgp.j2' %}
!
end
```

## Playbook: Offline Config Generation (`build_configs.yml`)

```yaml
---
- name: Render intended device configurations offline
  hosts: all
  gather_facts: false
  connection: local

  tasks:
    - name: Ensure intended config directory exists
      ansible.builtin.file:
        path: "intended/configs"
        state: directory
        mode: "0755"
      run_once: true

    - name: Render vendor-specific intended configuration
      ansible.builtin.template:
        src: "templates/{{ platform_dir }}/main.j2"
        dest: "intended/configs/{{ inventory_hostname }}.cfg"
        mode: "0644"
        lstrip_blocks: true
        trim_blocks: true
      vars:
        platform_dir: "{{ ansible_network_os.split('.')[-1] }}"
```

## Pushing Rendered Configs: Merge vs. Replace

| Mode | IOS-XE Mechanism | Junos Mechanism | When to Use |
| :--- | :--- | :--- | :--- |
| **Merge** | `ios_config` (`src:`) | `junos_config` (`update: merge`) | Brownfield networks where Ansible owns specific sections (NTP, VLANs, BGP) |
| **Replace** | `ios_config` (`replace: config` / `configure replace`) | `junos_config` (`update: override`) | Greenfield or fully modeled networks where Ansible owns 100% of the running config |

```yaml
- name: Deploy pre-rendered intended configuration to Cisco IOS-XE
  cisco.ios.ios_config:
    src: "intended/configs/{{ inventory_hostname }}.cfg"
    diff_against: running
    backup: true
    backup_options:
      dir_path: "backups/pre-deploy"
    save_when: modified
```

> [!IMPORTANT]
> Always run `build_configs.yml` and review `git diff intended/configs/` in a pull request before running `deploy_configs.yml`. Reviewers can inspect the exact CLI lines that will land on every router before a single packet is sent to production.
