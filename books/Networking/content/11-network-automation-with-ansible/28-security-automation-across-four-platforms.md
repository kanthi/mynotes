# Chapter 28 — Security Automation Across Four Platforms

## One Security Intent, Four Vendor Syntaxes

When a SOC incident requires blocking a malicious C2 prefix (`203.0.113.0/24`) or opening an approved application flow across your perimeter, your infrastructure rarely comes from a single vendor. Modeling security rules in a **vendor-neutral data schema** lets one Git commit deploy consistently across Cisco IOS-XE, Cisco ASA, Juniper SRX (Junos), and Fortinet FortiOS.

## Vendor-Neutral Security Intent (`group_vars/all/security_rules.yml`)

```yaml
---
emergency_blocklist:
  - name: "SOC_BLOCK_2026_001"
    cidr: "203.0.113.0/24"
    reason: "Confirmed C2 infrastructure - INC009142"

approved_app_flows:
  - id: 210
    name: "APP_BILLING_HTTPS"
    src_cidr: "10.20.0.0/16"
    dst_cidr: "10.90.10.0/24"
    proto: tcp
    port: 443
    action: permit
```

## Cross-Platform Implementation Matrix

| Platform | Collection & Module | Object / ACL Strategy |
| :--- | :--- | :--- |
| **Cisco IOS-XE** | `cisco.ios.ios_acls` | Named extended IPv4 ACLs with explicit sequence numbers and `log` |
| **Cisco ASA** | `cisco.asa.asa_ogs` + `cisco.asa.asa_acls` | Network/service `object-group`s referenced by interface `access-group` ACLs |
| **Juniper SRX (Junos)** | `junipernetworks.junos.junos_security_policies` | Zone-based address-book entries + security policies with `commit confirmed` |
| **Fortinet FortiOS** | `fortinet.fortios.fortios_firewall_address` + `fortios_firewall_policy` | Reusable address objects + ordered VDOM firewall policies via `httpapi` |

## Unified Multi-Platform Blocklist Playbook

```yaml
---
- name: Deploy SOC emergency blocklist across all perimeter platforms
  hosts: perimeter_security
  gather_facts: false

  tasks:
    - name: Block prefixes on Cisco IOS-XE perimeter routers
      cisco.ios.ios_acls:
        config:
          - afi: ipv4
            acls:
              - name: PERIMETER_IN
                acl_type: extended
                aces:
                  - sequence: 10
                    grant: deny
                    protocol: ip
                    source:
                      address: "{{ item.cidr | ansible.utils.ipaddr('network') }}"
                      wildcard_bits: "{{ item.cidr | ansible.utils.ipaddr('hostmask') }}"
                    destination:
                      any: true
                    log:
                      set: true
        state: merged
      loop: "{{ emergency_blocklist }}"
      when: ansible_network_os == 'cisco.ios.ios'

    - name: Create FortiGate blocklist address objects
      fortinet.fortios.fortios_firewall_address:
        vdom: "root"
        state: present
        firewall_address:
          name: "{{ item.name }}"
          type: "ipmask"
          subnet: "{{ item.cidr | ansible.utils.ipaddr('network') }} {{ item.cidr | ansible.utils.ipaddr('netmask') }}"
          comment: "{{ item.reason }}"
      loop: "{{ emergency_blocklist }}"
      when: ansible_network_os == 'fortinet.fortios.fortios'
```

> [!IMPORTANT]
> Always include an automated **policy sanity check** before deploying ACLs—verifying that your management bastion subnet (`10.255.0.0/24`) is explicitly permitted above any deny rules so automation can never lock itself out of a perimeter firewall.
