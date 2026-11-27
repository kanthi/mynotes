# Chapter 25 — Troubleshooting and Validation Playbooks

## Turning Tribal Knowledge into Executable Validation

Every senior network engineer has a mental checklist they run after a change: *Are all BGP sessions up? Did OSPF neighbor count drop? Are there interface CRC errors? Can we ping the core loopback?* Encoding those checks into Ansible validation playbooks turns tribal knowledge into repeatable, auditable pre/post-change gates.

## Pre-Change vs. Post-Change State Diffing

Capture structured operational state *before* a change (`--tags pre`), execute the change, then capture *after* (`--tags post`) and compare automatically:

```yaml
---
- name: Operational state validation for routing and interfaces
  hosts: edge_routers
  gather_facts: false

  tasks:
    - name: Parse BGP summary into structured facts
      ansible.utils.cli_parse:
        command: "show ip bgp summary"
        parser:
          name: ansible.netcommon.ntc_templates
        set_fact: bgp_summary

    - name: Parse OSPF neighbors into structured facts
      ansible.utils.cli_parse:
        command: "show ip ospf neighbor"
        parser:
          name: ansible.netcommon.ntc_templates
        set_fact: ospf_neighbors

    - name: Assert minimum healthy peer counts
      ansible.builtin.assert:
        that:
          - (bgp_summary | selectattr('state_pfxrcd', 'match', '^[0-9]+$') | list | length) >= expected_bgp_peers
          - (ospf_neighbors | selectattr('state', 'search', 'FULL') | list | length) >= expected_ospf_peers
        fail_msg: >-
          Control-plane health check failed on {{ inventory_hostname }}:
          BGP={{ bgp_summary | length }}, OSPF={{ ospf_neighbors | length }}
```

## Active Reachability Testing with `ios_ping`

Verify data-plane reachability across VRFs or loopbacks directly from the router itself:

```yaml
- name: Verify data-plane reachability to core DNS and WAN gateway
  cisco.ios.ios_ping:
    dest: "{{ item.dest }}"
    vrf: "{{ item.vrf | default(omit) }}"
    source: "{{ loopback0_ip }}"
    count: 5
    state: present
  loop:
    - { dest: "192.0.2.53", vrf: "CORP" }
    - { dest: "198.51.100.1" }
  loop_control:
    label: "{{ item.dest }} (vrf={{ item.vrf | default('global') }})"
```

## Incident Triage Bundle Playbook

During a P1 incident, running a single read-only triage playbook across 20 affected devices collects `show logging`, `show processes cpu history`, `show interfaces counters errors`, and `show bgp all summary` in under 15 seconds—saving every output to `triage/<incident_id>/<hostname>.txt` for immediate team analysis.
