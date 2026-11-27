# Appendix B — Module Cross-Reference

## Multi-Vendor Network Module Mapping

Use this table to translate common network automation tasks across Cisco IOS-XE, Cisco NX-OS, Juniper Junos, Fortinet FortiOS, and Arista EOS.

| Capability | Cisco IOS-XE (`cisco.ios`) | Cisco NX-OS (`cisco.nxos`) | Juniper Junos (`junipernetworks.junos`) | Fortinet FortiOS (`fortinet.fortios`) | Arista EOS (`arista.eos`) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Connection** | `network_cli` | `network_cli` / `httpapi` | `netconf` | `httpapi` | `network_cli` / `httpapi` |
| **Run CLI command** | `ios_command` | `nxos_command` | `junos_command` | `fortios_monitor` | `eos_command` |
| **Raw config / template** | `ios_config` | `nxos_config` | `junos_config` | `fortios_configuration_fact` | `eos_config` |
| **Gather facts** | `ios_facts` | `nxos_facts` | `junos_facts` | `fortios_configuration_fact` | `eos_facts` |
| **Interfaces (base)** | `ios_interfaces` | `nxos_interfaces` | `junos_interfaces` | `fortios_system_interface` | `eos_interfaces` |
| **L2 Switchports** | `ios_l2_interfaces` | `nxos_l2_interfaces` | `junos_l2_interfaces` | `fortios_switch_controller_managed_switch` | `eos_l2_interfaces` |
| **L3 IP Interfaces** | `ios_l3_interfaces` | `nxos_l3_interfaces` | `junos_l3_interfaces` | `fortios_system_interface` | `eos_l3_interfaces` |
| **VLANs** | `ios_vlans` | `nxos_vlans` | `junos_vlans` | `fortios_system_interface` | `eos_vlans` |
| **LACP / LAG** | `ios_lag_interfaces` | `nxos_lag_interfaces` | `junos_lag_interfaces` | `fortios_system_interface` | `eos_lag_interfaces` |
| **Static Routes** | `ios_static_routes` | `nxos_static_routes` | `junos_static_routes` | `fortios_router_static` | `eos_static_routes` |
| **OSPFv2** | `ios_ospfv2` | `nxos_ospfv2` | `junos_ospfv2` | `fortios_router_ospf` | `eos_ospfv2` |
| **BGP Global** | `ios_bgp_global` | `nxos_bgp_global` | `junos_bgp_global` | `fortios_router_bgp` | `eos_bgp_global` |
| **BGP Address Family** | `ios_bgp_address_family` | `nxos_bgp_address_family` | `junos_bgp_address_family` | `fortios_router_bgp` | `eos_bgp_address_family` |
| **ACLs / Policies** | `ios_acls` | `nxos_acls` | `junos_acls` / `junos_security_policies` | `fortios_firewall_policy` | `eos_acls` |
| **NTP** | `ios_ntp_global` | `nxos_ntp_global` | `junos_ntp_global` | `fortios_system_ntp` | `eos_ntp_global` |
| **Logging / Syslog** | `ios_logging_global` | `nxos_logging_global` | `junos_logging_global` | `fortios_log_syslogd_setting` | `eos_logging_global` |
| **Ping / Reachability** | `ios_ping` | `nxos_ping` | `junos_ping` | `fortios_monitor` | `eos_command` |

## Cross-Platform Utility Modules (`ansible.netcommon` & `ansible.utils`)

| Module FQCN | Purpose |
| :--- | :--- |
| `ansible.netcommon.cli_command` | OS-agnostic CLI command execution over `network_cli` |
| `ansible.netcommon.cli_config` | OS-agnostic CLI configuration push with diff/rollback |
| `ansible.netcommon.net_put` / `net_get` | SCP/SFTP firmware and file transfer to/from network devices |
| `ansible.utils.cli_parse` | Parse raw CLI output using TextFSM (`ntc_templates`), Genie, TTP, or XML |
| `ansible.utils.validate` | Validate YAML/JSON variable models against JSON Schema |
