# Chapter 17 — Parsing Device Output

## Why Raw Text Breaks Operational Playbooks

Running `show ip bgp summary` returns a human-readable ASCII table. Trying to validate neighbor counts or prefix counts with fragile `regex_search` breaks the moment a column wraps onto two lines. Structured parsing converts unstructured CLI output into clean JSON lists and dictionaries.

## Choosing a Parser Engine (`ansible.utils.cli_parse`)

The `ansible.utils.cli_parse` module provides a unified interface to four industry-standard parsers:

| Parser | Best For | Pros / Cons |
| :--- | :--- | :--- |
| **TextFSM + NTC-Templates** | Broad multi-vendor `show` commands (IOS, NX-OS, Junos, EOS, ASA) | Huge community template library (`ntc-templates`); simple state-machine syntax |
| **Cisco Genie / pyATS** | Deep Cisco IOS/IOS-XE/NX-OS/IOS-XR operational state | Rich nested schemas (BGP, OSPF, interfaces, routing tables); heavier Python dependency |
| **TTP (Template Text Parser)** | Custom or semi-structured configs and logs | Templates look like the output itself with `{{ variable }}` placeholders; fast to write |
| **XML / JSON native** | Junos (`| display json`), Arista EOS (`| json`), NX-OS (`| json`) | Zero parsing fragility—always prefer native JSON/XML when the NOS supports it |

## Parsing `show ip bgp summary` with `cli_parse` and TextFSM

```yaml
- name: Run and parse BGP summary with NTC-Templates TextFSM
  ansible.utils.cli_parse:
    command: "show ip bgp summary"
    parser:
      name: ansible.netcommon.ntc_templates
    set_fact: bgp_peers

- name: Assert every BGP peer is established with at least 1 prefix
  ansible.builtin.assert:
    that:
      - item.state_pfxrcd is match('^[0-9]+$')
      - (item.state_pfxrcd | int) >= 1
    fail_msg: "BGP peer {{ item.bgp_neigh }} on {{ inventory_hostname }} is in state {{ item.state_pfxrcd }}"
  loop: "{{ bgp_peers }}"
  loop_control:
    label: "{{ item.bgp_neigh }} (AS {{ item.neigh_as }})"
```

## Parsing Custom Output with TTP

When no NTC template exists for a vendor command, write a tiny TTP template (`templates/ttp/ios_cdp.ttp`):

```text
Device ID: {{ neighbor }}
  IP address: {{ mgmt_ip }}
Interface: {{ local_intf }},  Port ID (outgoing port): {{ remote_intf }}
```

And invoke it inline:

```yaml
- name: Parse CDP neighbors detail with TTP
  ansible.utils.cli_parse:
    command: "show cdp neighbors detail"
    parser:
      name: ansible.utils.ttp
      template_path: "templates/ttp/ios_cdp.ttp"
    set_fact: cdp_neighbors
```

## Native JSON Parsing (Zero External Templates)

On Arista EOS, NX-OS, or Junos, request structured JSON directly from the device:

```yaml
- name: Gather native JSON BGP summary on Arista EOS
  arista.eos.eos_command:
    commands:
      - show ip bgp summary | json
  register: eos_bgp

- name: Inspect parsed peer dictionary
  ansible.builtin.debug:
    msg: "{{ eos_bgp.stdout[0].vrfs.default.peers | dict2items }}"
```

Whenever a device supports native JSON or NETCONF XML, prefer it over screen-scraping—your validation playbooks will never break on whitespace changes across OS upgrades.
