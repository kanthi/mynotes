# Chapter 21 — Golden Configurations and Compliance

## What Is a Golden Configuration?

A **golden configuration** is the authoritative baseline every device of a given role must satisfy—covering security hardening (SSHv2 only, no HTTP server, AAA/TACACS+, SNMPv3 with AuthPriv), management services (NTP, syslog, banners), and architectural invariants (BPDU guard on access ports, uRPF on edge links).

## Continuous Compliance Checking Without Making Changes

The safest way to introduce golden configs in a brownfield enterprise is **read-only drift detection**. Run compliance checks nightly in `--check --diff` mode (or compare against `intended/configs/`) and publish a structured compliance report.

```yaml
---
- name: Audit Cisco IOS-XE security baseline compliance
  hosts: ios
  gather_facts: false

  tasks:
    - name: Compare live running-config against golden security baseline
      cisco.ios.ios_config:
        src: "templates/ios/golden_security.j2"
        diff_against: running
      check_mode: true
      diff: true
      register: compliance_result

    - name: Record non-compliant host findings
      ansible.builtin.set_fact:
        is_compliant: "{{ not compliance_result.changed }}"
        missing_lines: "{{ compliance_result.updates | default([]) }}"

    - name: Generate Markdown compliance report on control node
      ansible.builtin.template:
        src: "templates/reports/compliance_report.md.j2"
        dest: "reports/compliance_{{ lookup('pipe', 'date +%Y-%m-%d') }}.md"
        mode: "0644"
      delegate_to: localhost
      run_once: true
```

## Validating Negative Rules ("Must Not Exist")

Template comparison catches *missing* lines, but security audits also require catching *prohibited* legacy lines (such as `ip http server`, `snmp-server community public`, or `transport input telnet`). Fetch the running config and assert forbidden patterns explicitly:

```yaml
- name: Gather running configuration for negative security assertions
  cisco.ios.ios_command:
    commands:
      - show running-config
  register: running_cfg

- name: Check for prohibited insecure services
  ansible.builtin.set_fact:
    security_violations: >-
      {{
        prohibited_patterns
        | select('in', running_cfg.stdout[0])
        | list
      }}
  vars:
    prohibited_patterns:
      - "ip http server"
      - "transport input telnet"
      - "snmp-server community public"
      - "service finger"
```

## Remediation Workflow: Report → Approve → Enforce

1. **Audit (nightly):** Detect drift and prohibited config lines across the fleet; post a summary table to Slack/Jira.
2. **Triage:** Determine whether drift was an unauthorized manual hack (remediate) or a legitimate emergency fix that must be back-ported into the Source of Truth.
3. **Remediate (change window):** Re-run the exact same golden-config playbook *without* `check_mode: true` during an approved window to bring non-compliant devices back into alignment.
