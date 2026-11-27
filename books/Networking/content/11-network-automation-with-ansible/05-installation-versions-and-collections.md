# Chapter 5 — Installation, Versions and Collections

## `ansible-core` vs. the `ansible` Community Package

Modern Ansible is split into two distinct packages:

| Package | What It Contains | When to Use |
| :--- | :--- | :--- |
| **`ansible-core`** | The engine (`ansible-playbook`, `ansible-galaxy`, `ansible-vault`), built-in modules (`ansible.builtin`), and core plugins | **Recommended for production.** Pin exact versions and install only the vendor collections you need |
| **`ansible`** | `ansible-core` bundled with 100+ community collections (~500 MB+) | Quick personal sandboxes where disk space and dependency pinning do not matter |

## Clean Virtual Environment Installation

Never install Ansible into your operating system's global Python (`sudo pip install`). Isolate every project in a dedicated Python virtual environment:

```bash
# Create and activate an isolated virtual environment
python3 -m venv ~/.venvs/netops-ansible
source ~/.venvs/netops-ansible/bin/activate

# Install pinned ansible-core and essential network libraries
pip install --upgrade pip
pip install "ansible-core>=2.17,<2.19" paramiko netaddr jmespath textfsm ntc-templates pyats genie
```

Verify the installation and Python runtime paths:

```bash
ansible --version
```

```text
ansible [core 2.17.6]
  config file = /Users/netops/net-automation/ansible.cfg
  configured module search path = ['/Users/netops/.ansible/plugins/modules']
  ansible python module location = /Users/netops/.venvs/netops-ansible/lib/python3.12/site-packages/ansible
  python version = 3.12.7
  jinja version = 3.1.4
  libyaml = True
```

> [!NOTE]
> Confirm `libyaml = True` in `ansible --version`. Without the C-backed PyYAML (`libyaml`), parsing large inventories and host variable files is noticeably slower.

## Managing Vendor Collections with `requirements.yml`

Network modules live in **Ansible Collections** (`namespace.collection`), distributed via Ansible Galaxy or Red Hat Automation Hub. Declare every required collection and its pinned version in `collections/requirements.yml`:

```yaml
---
collections:
  - name: ansible.netcommon
    version: ">=7.0.0,<8.0.0"
  - name: ansible.utils
    version: ">=5.0.0,<6.0.0"
  - name: cisco.ios
    version: ">=9.0.0,<10.0.0"
  - name: cisco.nxos
    version: ">=9.0.0,<10.0.0"
  - name: cisco.asa
    version: ">=6.0.0,<7.0.0"
  - name: junipernetworks.junos
    version: ">=9.0.0,<10.0.0"
  - name: fortinet.fortios
    version: ">=2.3.0,<3.0.0"
  - name: arista.eos
    version: ">=10.0.0,<11.0.0"
```

Install all collections reproducibly:

```bash
ansible-galaxy collection install -r collections/requirements.yml --force-with-deps
ansible-galaxy collection list
```

## Project-Local `ansible.cfg`

Always place an `ansible.cfg` at the root of your repository so every team member runs with identical settings:

```ini
[defaults]
inventory = inventory/hosts.yml
collections_path = ./collections:~/.ansible/collections
roles_path = ./roles
stdout_callback = ansible.builtin.default
result_format = yaml
retry_files_enabled = False
gathering = explicit
forks = 20
timeout = 30

[persistent_connection]
connect_timeout = 30
command_timeout = 60
```

> [!IMPORTANT]
> Setting `gathering = explicit` prevents Ansible from trying to run Linux `setup` facts against network devices at the start of every play. Always gather network facts explicitly using `cisco.ios.ios_facts`, `junipernetworks.junos.junos_facts`, or `gather_facts: true` only when `ansible_network_os` is properly mapped.

## Reproducibility: Moving Toward Execution Environments

As your team grows, even Python virtual environments drift between laptops. In [Chapter 32 — AWX and Ansible Automation Platform](32-awx-and-ansible-automation-platform.md), you will package `ansible-core`, Python libraries, and collections into an immutable container image called an **Execution Environment (EE)** built with `ansible-builder`.
