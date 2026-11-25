---
title: '03. Project: Automated 3-Node Proxmox VE + Ceph + Ansible Cluster'
---

# Project: Automated 3-Node Proxmox VE + Ceph + Ansible Cluster

Provision a 3-node **Proxmox VE / KVM** cluster (bare metal or nested VMs) backed by **Ceph** replicated storage, fully automated via **Ansible** for quorum networking, SSH/PKI setup, automated `vzdump`/PBS backups, and Prometheus/Zabbix alerting.

## Cluster Topology

```text
3 Nodes: pve-1, pve-2, pve-3 (Corosync quorum + dedicated Ceph replication VLAN)
   ├── Ceph MON + MGR + OSDs (size=3, min_size=2 for zero-data-loss single-node failure)
   ├── Ansible Roles: base-hardening, ssh-ca, ceph-osd, backup-schedule, node_exporter
   └── Monitoring: Prometheus pve_exporter + Ceph mgr prometheus module -> Grafana + Alertmanager
```

## Acceptance Drills

- Run `ceph health detail` and confirm `HEALTH_OK` with all PGs `active+clean`.
- Power off `pve-3` abruptly; verify HA-managed VMs live-failover to `pve-1`/`pve-2` and Ceph remains readable/writable (`min_size=2`).
- Restore a VM from an automated backup onto a fresh VMID and verify boot integrity.
