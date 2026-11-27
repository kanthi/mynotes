# Chapter 26 — Campus: 802.1X, VLANs and Access Switching

## The Campus Automation Opportunity

Campus access switches are the highest-volume devices in most enterprises—hundreds of identical Catalyst or EX switches with thousands of user, AP, printer, and VoIP ports. Standardizing port profiles and 802.1X deployment with Ansible eliminates years of hand-crafted port drift.

## Modeling Campus Port Profiles in YAML

Define reusable port-profile schemas in `group_vars/campus_access.yml` so individual switch files only map interface ranges to profile names:

```yaml
# group_vars/campus_access.yml
campus_vlans:
  - { vlan_id: 10, name: CORP_DATA }
  - { vlan_id: 20, name: CORP_VOICE }
  - { vlan_id: 30, name: WIFI_MGMT }
  - { vlan_id: 99, name: NET_MGMT }
  - { vlan_id: 999, name: BLACKHOLE }

dot1x_mode: monitor   # monitor (low-impact) -> closed (enforced)

# host_vars/sw-bldg1-fl2.yml
switch_ports:
  user_dot1x:
    - GigabitEthernet1/0/1
    - GigabitEthernet1/0/2
  wireless_ap:
    - GigabitEthernet1/0/24
  uplinks:
    - TenGigabitEthernet1/1/1
    - TenGigabitEthernet1/1/2
```

## Staged 802.1X Rollout (Monitor → Low-Impact → Closed)

Deploying 802.1X in `closed` mode overnight causes helpdesk chaos. Drive the deployment mode from a single group/host variable (`dot1x_mode`) inside your Jinja2 port template (`templates/ios/campus_dot1x_port.j2`):

```jinja2
{% for intf in switch_ports.user_dot1x %}
interface {{ intf }}
 description USER_ACCESS_DOT1X
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 20
 spanning-tree portfast
 spanning-tree bpduguard enable
 authentication host-mode multi-domain
 authentication order dot1x mab
 authentication priority dot1x mab
 authentication port-control auto
{% if dot1x_mode == 'monitor' %}
 authentication open
{% elif dot1x_mode == 'closed' %}
 no authentication open
{% endif %}
 mab
 dot1x pae authenticator
 dot1x timeout tx-period 10
!
{% endfor %}
```

## Enforcing VLANs, Trunks, and Access Ports Together

```yaml
- name: Deploy campus VLAN database
  cisco.ios.ios_vlans:
    config: "{{ campus_vlans }}"
    state: merged

- name: Configure uplinks with explicit allowed VLAN list
  cisco.ios.ios_l2_interfaces:
    config:
      - name: "{{ item }}"
        mode: trunk
        trunk:
          native_vlan: 999
          allowed_vlans:
            - "10"
            - "20"
            - "30"
            - "99"
    state: replaced
  loop: "{{ switch_ports.uplinks }}"

- name: Apply staged 802.1X port templates
  cisco.ios.ios_config:
    src: templates/ios/campus_dot1x_port.j2
    save_when: modified
```

When a floor completes its 2-week `monitor` audit with zero unauthorized devices in RADIUS logs, promoting that switch to full enforcement is a single-line Git PR changing `dot1x_mode: closed`.
