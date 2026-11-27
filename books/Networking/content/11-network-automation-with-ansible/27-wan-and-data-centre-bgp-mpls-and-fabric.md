# Chapter 27 — WAN and Data Centre: BGP, MPLS and Fabric

## Why Fabric and WAN Configs Must Be Data-Driven

In a Clos EVPN-VXLAN data centre or an MPLS/BGP WAN, every link has two endpoints that must agree on IP subnets, ASNs, MTU, BFD timers, and route-targets. Configuring peers manually invites asymmetric policies and silent black holes. Modeling the topology once in YAML lets Ansible generate both sides consistently.

## Data-Driven eBGP Underlay & EVPN Overlay Model

```yaml
# host_vars/leaf-01.yml
bgp:
  asn: 65101
  router_id: 10.255.0.11
  underlay_peers:
    - { ip: 172.16.1.0, asn: 65001, desc: "spine-01" }
    - { ip: 172.16.1.2, asn: 65002, desc: "spine-02" }
  evpn_peers:
    - { ip: 10.255.0.1, asn: 65001, desc: "spine-01-evpn" }
    - { ip: 10.255.0.2, asn: 65002, desc: "spine-02-evpn" }
  redistribute_loopbacks:
    - 10.255.0.11/32
    - 10.255.1.11/32
```

## Deploying Structured BGP with Resource Modules (`cisco.nxos`)

```yaml
- name: Configure NX-OS BGP global process and eBGP underlay peers
  cisco.nxos.nxos_bgp_global:
    config:
      as_number: "{{ bgp.asn | string }}"
      router_id: "{{ bgp.router_id }}"
      bestpath:
        as_path:
          multipath_relax: true
      neighbors: "{{ bgp_neighbor_list }}"
    state: merged
  vars:
    bgp_neighbor_list: >-
      {%- set ns = namespace(items=[]) -%}
      {%- for p in bgp.underlay_peers -%}
      {%- set _ = ns.items.append({
            'neighbor_address': p.ip,
            'remote_as': p.asn | string,
            'description': p.desc,
            'bfd': 'singlehop'
          }) -%}
      {%- endfor -%}
      {{ ns.items }}
```

## Safe BGP Maintenance Drain Playbook

Before upgrading or replacing a WAN edge router or DC spine, drain traffic cleanly using **BGP Graceful Shutdown** (`graceful-shutdown` community `65535:0`, RFC 8326) rather than pulling fibers:

```yaml
---
- name: Gracefully drain BGP traffic from target WAN/DC router
  hosts: "{{ target_router }}"
  gather_facts: false

  tasks:
    - name: Apply gshut route-map to eBGP peers
      cisco.ios.ios_config:
        parents:
          - "router bgp {{ bgp.asn }}"
          - "address-family ipv4 unicast"
        lines:
          - "neighbor {{ item.ip }} route-map BGP_GSHUT_OUT out"
      loop: "{{ bgp.wan_peers }}"

    - name: Wait 30 seconds for global BGP reconvergence
      ansible.builtin.pause:
        seconds: 30

    - name: Verify interface packet rate dropped near zero before maintenance
      cisco.ios.ios_command:
        commands:
          - "show interfaces {{ item.intf }} | include rate"
      loop: "{{ bgp.wan_peers }}"
      register: traffic_rates
```
