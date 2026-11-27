# Chapter 23 — Facts, Reports and Documentation

## Eliminating Stale Network Documentation

Manual Excel spreadsheets and Visio diagrams are obsolete the afternoon they are written. When Ansible collects live facts (`ios_facts`, `junos_facts`, `eos_facts`) and renders them through Jinja2 templates, your asset inventory, OS version matrix, and interface documentation stay permanently synchronized with production reality.

## Multi-Vendor Asset & OS Audit Playbook

```yaml
---
- name: Gather fleet facts and generate Markdown + CSV inventory reports
  hosts: all
  gather_facts: false

  tasks:
    - name: Gather Cisco IOS-XE facts
      cisco.ios.ios_facts:
        gather_subset: [min, hardware, interfaces]
      when: ansible_network_os == 'cisco.ios.ios'

    - name: Gather Juniper Junos facts
      junipernetworks.junos.junos_facts:
        gather_subset: [min, hardware, interfaces]
      when: ansible_network_os == 'junipernetworks.junos.junos'

    - name: Render fleet inventory CSV report
      ansible.builtin.template:
        src: templates/reports/asset_inventory.csv.j2
        dest: "reports/network_assets_{{ lookup('pipe', 'date +%Y-%m-%d') }}.csv"
        mode: "0644"
      delegate_to: localhost
      run_once: true

    - name: Render Markdown executive summary
      ansible.builtin.template:
        src: templates/reports/fleet_summary.md.j2
        dest: "reports/fleet_summary_{{ lookup('pipe', 'date +%Y-%m-%d') }}.md"
        mode: "0644"
      delegate_to: localhost
      run_once: true
```

## Example Report Template (`templates/reports/asset_inventory.csv.j2`)

```jinja2
Hostname,Site,Platform,Model,Serial Number,OS Version,Mgmt IP
{% for host in groups['all'] | sort %}
{% set hv = hostvars[host] %}
{{ host }},{{ hv.site_code | default('N/A') }},{{ hv.ansible_network_os | default('unknown') }},{{ hv.ansible_net_model | default('N/A') }},{{ hv.ansible_net_serialnum | default('N/A') }},{{ hv.ansible_net_version | default('N/A') }},{{ hv.ansible_host | default('N/A') }}
{% endfor %}
```

## High-Value Operational Reports to Automate Weekly

- **PSIRT / EOL version exposure:** Flag every device running an IOS-XE or FortiOS version below your security baseline.
- **Unused port reclamation:** Identify access switchports that have been `notconnect` for >90 days.
- **LLDP/CDP neighbor drift:** Compare discovered physical neighbors against expected cabling in NetBox to catch undocumented patch-panel moves.
- **Certificate & license expiry:** Surface expiring TLS/IPsec certificates and Smart Licensing compliance warnings 30 days ahead of failure.
