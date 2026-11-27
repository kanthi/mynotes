# Chapter 4 — Building Your Lab

## Why You Must Never Learn on Production

Every automation engineer eventually writes a playbook with a wrong variable scope, a missing `replace` boundary, or an inverted conditional. Discovering that mistake against a virtual topology takes 10 seconds to reset; discovering it on a production core router creates an incident bridge.

## Choosing a Lab Platform

Select a lab runtime based on your available hardware and target vendor NOS images:

| Platform | Runtime Model | Strengths | Resource Footprint |
| :--- | :--- | :--- | :--- |
| **Containerlab** | Linux containers + VRNetlab VMs | Git-versioned YAML topologies, instant startup, auto-generated `ansible-inventory.yml` | Light (2–8 GB RAM for FRR/cEOS/SRL; 16+ GB with VM NOS) |
| **netlab** | High-level topology compiler | Generates Containerlab/libvirt configs + Ansible inventory + base OSPF/BGP automatically | Moderate |
| **EVE-NG / GNS3** | Full KVM/QEMU hypervisor | Familiar GUI, broad Cisco IOSv/CSR1000v/NX-OSv/FortiGate VM support | Heavy (16–64 GB RAM) |
| **Cisco CML** | Official Cisco virtual lab | Officially licensed IOSv, IOS-XE (Cat8000v), NX-OS 9000v, ASAv images | Heavy (16–32 GB RAM) |

> [!TIP]
> On macOS Apple Silicon or Linux, **Containerlab** (via OrbStack on macOS or native Docker on Linux) is the fastest way to spin up reproducible multi-router labs. See [Lab Platform](../01-lab-platform/index.md) and [Appendix C — Lab Build Guide](38-appendix-c-lab-build-guide.md) for step-by-step setup.

## Reference Multi-Vendor Lab Topology

Throughout this track, we use a clean management-connected topology with out-of-band (OOB) SSH/API access from the Ansible control node:

```text
                [ Ansible Control Node (172.20.20.1/24) ]
                                    |
                     OOB Management Network (172.20.20.0/24)
            +---------------+-------+-------+---------------+
            |               |               |               |
       172.20.20.11    172.20.20.12    172.20.20.13    172.20.20.14
     [ rtr-edge-01 ] [ sw-core-01 ]  [ rtr-wan-01 ]  [ fw-edge-01 ]
      (IOS-XE/cEOS)    (IOS-XE/FRR)    (Junos/FRR)   (FortiOS/Linux)
            \               |               /
             +--- Data-plane links (OSPF / BGP / VLANs) ---+
```

## Example Containerlab Definition (`lab.clab.yml`)

Here is a minimal, free-image Containerlab topology using Arista cEOS or FRR containers that boots in seconds and generates an Ansible inventory automatically:

```yaml
name: ansible-netlab
mgmt:
  network: ansible-mgmt
  ipv4-subnet: 172.20.20.0/24

topology:
  nodes:
    rtr1:
      kind: ceos
      image: ceos:4.32.0F
      mgmt-ipv4: 172.20.20.11
      labels:
        ansible-group: routers
    rtr2:
      kind: ceos
      image: ceos:4.32.0F
      mgmt-ipv4: 172.20.20.12
      labels:
        ansible-group: routers
    sw1:
      kind: ceos
      image: ceos:4.32.0F
      mgmt-ipv4: 172.20.20.13
      labels:
        ansible-group: switches

  links:
    - endpoints: ["rtr1:eth1", "rtr2:eth1"]
    - endpoints: ["rtr1:eth2", "sw1:eth1"]
    - endpoints: ["rtr2:eth2", "sw1:eth2"]
```

## Deploying and Verifying Management Reachability

```bash
# Deploy the topology
sudo containerlab deploy -t lab.clab.yml

# Inspect assigned management IPs and generated Ansible inventory
sudo containerlab inspect -t lab.clab.yml
ls -la clab-ansible-netlab/ansible-inventory.yml

# Verify SSH reachability to the first node
ssh admin@172.20.20.11 "show version"
```

## Lab Hygiene Rules

- **Snapshot or version-control startup configs:** Always keep a clean baseline config directory so you can `containerlab destroy && containerlab deploy` back to a known state in under 30 seconds.
- **Dedicated automation credentials:** Create a dedicated `netops` / `ansible` account on lab devices with privilege level 15 (or equivalent superuser role) rather than reusing personal logins.
- **No host-key friction in ephemeral labs:** Containerlab regenerates SSH host keys on rebuild; disable strict host key checking *only* inside your lab `ansible.cfg` (`host_key_checking = False`), never in production.
