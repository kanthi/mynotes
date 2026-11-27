# Appendix C — Lab Build Guide

## Quick-Start Containerlab + Ansible Environment

This guide builds a self-contained, reproducible network automation lab on Linux or macOS (via OrbStack/Docker) in under 10 minutes. For host-level setup details, see [Lab Platform](../01-lab-platform/index.md).

## Step 1: Prepare the Control Node Python Environment

```bash
mkdir -p ~/netops-lab && cd ~/netops-lab
python3 -m venv .venv
source .venv/bin/activate

pip install --upgrade pip
pip install "ansible-core>=2.17,<2.19" ansible-pylibssh paramiko netaddr jmespath textfsm ntc-templates yamllint ansible-lint

ansible-galaxy collection install \
  ansible.netcommon ansible.utils cisco.ios cisco.nxos \
  junipernetworks.junos fortinet.fortios arista.eos
```

## Step 2: Create a Free-Image Multi-Router Topology (`lab.clab.yml`)

This topology uses freely available FRRouting (`quay.io/frrouting/frr`) or Arista cEOS containers with an out-of-band management network:

```yaml
name: netops-ansible-lab
mgmt:
  network: netops-mgmt
  ipv4-subnet: 172.20.20.0/24

topology:
  nodes:
    rtr1:
      kind: linux
      image: quay.io/frrouting/frr:10.1.1
      mgmt-ipv4: 172.20.20.11
    rtr2:
      kind: linux
      image: quay.io/frrouting/frr:10.1.1
      mgmt-ipv4: 172.20.20.12
    sw1:
      kind: linux
      image: quay.io/frrouting/frr:10.1.1
      mgmt-ipv4: 172.20.20.13

  links:
    - endpoints: ["rtr1:eth1", "rtr2:eth1"]
    - endpoints: ["rtr1:eth2", "sw1:eth1"]
    - endpoints: ["rtr2:eth2", "sw1:eth2"]
```

> [!TIP]
> If you have licensed Cisco IOS-XE (Cat8000v), NX-OS 9000v, Juniper vJunos-router, or FortiGate VM images, wrap them with **VRNetlab** so Containerlab deploys them identically alongside containers and still generates `ansible-inventory.yml` automatically.

## Step 3: Deploy, Verify, and Run Your First Ad-Hoc Check

```bash
# Deploy topology
sudo containerlab deploy -t lab.clab.yml

# Verify generated inventory
ansible-inventory -i clab-netops-ansible-lab/ansible-inventory.yml --graph

# Test Ansible connectivity against all lab nodes
ansible all -i clab-netops-ansible-lab/ansible-inventory.yml -m ansible.builtin.ping
```

## Step 4: Clean Teardown

```bash
sudo containerlab destroy -t lab.clab.yml --cleanup
```
