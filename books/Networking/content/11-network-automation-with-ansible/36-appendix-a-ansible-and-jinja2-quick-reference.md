# Appendix A — Ansible and Jinja2 Quick Reference

## Essential `ansible-playbook` CLI Flags

| Command / Flag | Effect |
| :--- | :--- |
| `ansible-playbook site.yml --syntax-check` | Validate YAML and playbook syntax without connecting |
| `ansible-playbook site.yml --check --diff` | Dry-run mode with line-by-line configuration diffs |
| `ansible-playbook site.yml --limit rtr-edge-01` | Restrict execution to one host or host pattern |
| `ansible-playbook site.yml --tags "vlans,bgp"` | Run only tasks/roles matching listed tags |
| `ansible-playbook site.yml --skip-tags "save"` | Skip tasks matching listed tags |
| `ansible-playbook site.yml -e "target_vlan=40"` | Pass highest-precedence extra variables |
| `ansible-playbook site.yml --list-tasks` | Print every task that would run without executing |
| `ansible-playbook site.yml -vvv` | Detailed SSH / persistent connection troubleshooting |
| `ansible-inventory --graph` | Display inventory group hierarchy |
| `ansible-inventory --host <name>` | Dump all resolved variables for a single host |
| `ansible-doc cisco.ios.ios_vlans` | View built-in module parameters and examples offline |

## Jinja2 Control Structures & Filters

```jinja2
{# Conditionals #}
{% if intf.mode == 'trunk' %}
 switchport mode trunk
 switchport trunk allowed vlan {{ intf.vlans | join(',') }}
{% elif intf.mode == 'access' %}
 switchport mode access
 switchport access vlan {{ intf.vlan | default(10) }}
{% endif %}

{# Loops with loop metadata #}
{% for peer in bgp_peers | sort(attribute='ip') %}
! Peer {{ loop.index }} of {{ loop.length }}
 neighbor {{ peer.ip }} remote-as {{ peer.asn }}
{% endfor %}
```

### Network & Data Filters

| Filter | Example | Output |
| :--- | :--- | :--- |
| `default(val, true)` | `desc \| default('NONE', true)` | Fallback if undefined or empty string |
| `ansible.utils.ipaddr('address')` | `'10.1.1.1/30' \| ...` | `'10.1.1.1'` |
| `ansible.utils.ipaddr('netmask')` | `'10.1.1.1/30' \| ...` | `'255.255.255.252'` |
| `ansible.utils.ipaddr('hostmask')` | `'10.1.1.1/30' \| ...` | `'0.0.0.3'` |
| `ansible.utils.ipaddr('network/prefix')` | `'10.1.1.1/30' \| ...` | `'10.1.1.0/30'` |
| `selectattr('enabled', 'equalto', true)` | Filter list of dicts | Matching list items |
| `map(attribute='name') \| list` | Extract field from list of dicts | `['Gi0/1', 'Gi0/2']` |
| `dict2items` | Convert dict to `[{key, value}]` | Iterable list for `loop:` |
| `to_nice_yaml(indent=2)` | Serialize data structure | Clean formatted YAML |
| `regex_replace('^Gi', 'GigabitEthernet')` | Normalize interface names | `'GigabitEthernet0/1'` |
