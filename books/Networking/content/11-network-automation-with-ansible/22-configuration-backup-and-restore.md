# Chapter 22 — Configuration Backup and Restore

## Git-Backed Automated Configuration Backups

Storing router backups in a dated folder on a jump host is better than nothing, but committing cleaned backups into a **Git repository** gives you instant line-by-line historical diffs, blame/timestamps, and zero storage bloat across years of daily snapshots.

```yaml
---
- name: Daily multi-vendor configuration backup to Git
  hosts: all
  gather_facts: false

  tasks:
    - name: Backup Cisco IOS-XE running configuration
      cisco.ios.ios_config:
        backup: true
        backup_options:
          dir_path: "backups/{{ site_code | default('global') }}"
          filename: "{{ inventory_hostname }}.cfg"
      when: ansible_network_os == 'cisco.ios.ios'

    - name: Backup Juniper Junos configuration
      junipernetworks.junos.junos_config:
        backup: true
        backup_options:
          dir_path: "backups/{{ site_code | default('global') }}"
          filename: "{{ inventory_hostname }}.conf"
      when: ansible_network_os == 'junipernetworks.junos.junos'

    - name: Strip volatile clock/NTP timestamps to avoid noisy Git diffs
      ansible.builtin.lineinfile:
        path: "backups/{{ site_code | default('global') }}/{{ inventory_hostname }}.cfg"
        regexp: "^(! Last configuration change|! NVRAM config last updated|ntp clock-period)"
        state: absent
      delegate_to: localhost
      when: ansible_network_os == 'cisco.ios.ios'

    - name: Commit and push changed backups to Git
      ansible.builtin.shell: |
        git add backups/
        git diff --cached --quiet || git commit -m "Backup: $(date -u +%Y-%m-%dT%H:%MZ)"
      args:
        chdir: "{{ playbook_dir }}/.."
      delegate_to: localhost
      run_once: true
      changed_when: false
```

## Tested Restore Procedures

A backup you have never restored is an assumption, not a recovery plan. Know the native atomic restore mechanism per platform:

| Platform | Atomic Restore Mechanism | Key Requirement |
| :--- | :--- | :--- |
| **Cisco IOS-XE** | `configure replace flash:<file> force` | Requires `archive` enabled and valid full config syntax |
| **Cisco NX-OS** | `rollback running-config file bootflash:<file> atomic` | Best-in-class atomic rollback via checkpoints |
| **Juniper Junos** | `load override <file>` + `commit confirmed 5` | Built-in rollback archive (`rollback 1` .. `rollback 49`) |
| **Fortinet FortiOS** | `fortios_monitor` (`system.config.restore`) | Encrypted or plaintext full config restore triggers reboot |

> [!WARNING]
> On Cisco IOS-XE, `configure replace` aborts if a command cannot be cleanly reversed (such as removing a VRF still referenced by an interface). Always test restore playbooks against a virtual lab image matching your production OS version.
