# Chapter 16 — Beyond Cisco: Juniper and Fortinet

## Juniper Junos (`junipernetworks.junos`)

Junos was built from day one around a transactional XML datastore and NETCONF—making it one of the safest platforms in the industry to automate.

### Enabling NETCONF and Configuring Connection Variables

On Junos: `set system services netconf ssh`. On Ansible (`group_vars/junos.yml`):

```yaml
ansible_network_os: junipernetworks.junos.junos
ansible_connection: ansible.netcommon.netconf
ansible_port: 830
```

### Atomic Commits and `confirm` Rollback

Every `junos_config` or Junos resource module edits the **candidate configuration**, validates syntax before commit, and supports automatic rollback if the automation loses reachability:

```yaml
- name: Push Junos NTP and syslog config with 5-minute commit-confirmed safety net
  junipernetworks.junos.junos_config:
    lines:
      - set system ntp server 192.0.2.123
      - set system syslog host 192.0.2.50 any info
    comment: "CHG0042101: Standardize NTP and syslog"
    confirm: 5
    confirm_commit: false

- name: Verify control-plane reachability before confirming commit
  junipernetworks.junos.junos_command:
    commands:
      - show bgp summary
  register: bgp_verify

- name: Confirm commit permanently
  junipernetworks.junos.junos_config:
    confirm_commit: true
```

If the change accidentally breaks management SSH/NETCONF, Junos automatically rolls back to the previous configuration after 5 minutes—zero truck roll required.

## Fortinet FortiOS (`fortinet.fortios`)

FortiGate firewalls expose a comprehensive REST API managed via `ansible.netcommon.httpapi` and the `fortinet.fortios` collection.

### Connection Variables (`group_vars/fortios.yml`)

```yaml
ansible_network_os: fortinet.fortios.fortios
ansible_connection: ansible.netcommon.httpapi
ansible_httpapi_use_ssl: true
ansible_httpapi_validate_certs: true
ansible_httpapi_port: 443
ansible_fortios_access_token: "{{ vault_fortios_token }}"
```

### Managing Address Objects and Firewall Policies

FortiOS modules map 1-to-1 to CLI/API paths (`fortios_firewall_address`, `fortios_firewall_policy`, `fortios_router_static`):

```yaml
- name: Create corporate web subnet address object on FortiGate
  fortinet.fortios.fortios_firewall_address:
    vdom: "root"
    state: present
    firewall_address:
      name: "NET_CORP_WEB"
      type: "ipmask"
      subnet: "10.40.10.0 255.255.255.0"
      comment: "Managed by Ansible"

- name: Allow HTTPS from Corp Users to Corp Web
  fortinet.fortios.fortios_firewall_policy:
    vdom: "root"
    state: present
    firewall_policy:
      policyid: 101
      name: "ALLOW_CORP_TO_WEB_HTTPS"
      srcintf:
        - name: "port2"
      dstintf:
        - name: "port3"
      srcaddr:
        - name: "NET_CORP_USERS"
      dstaddr:
        - name: "NET_CORP_WEB"
      service:
        - name: "HTTPS"
      action: "accept"
      schedule: "always"
      logtraffic: "all"
```

> [!NOTE]
> Always specify explicit `vdom:` on FortiOS tasks and remember that FortiOS policy evaluation is top-down by sequence order—use `fortios_firewall_policy` carefully or manage ordering with `fortinet.fortios.fortios_firewall_policy_move`.
