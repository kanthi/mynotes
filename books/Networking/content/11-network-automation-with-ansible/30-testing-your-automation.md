# Chapter 30 — Testing Your Automation

## The Network Automation Testing Pyramid

Testing network automation is not optional—it is what gives teams the confidence to deploy during business hours instead of 3:00 AM Sundays. Layer four progressively deeper checks:

| Layer | Tools | Speed | What It Catches |
| :--- | :--- | :--- | :--- |
| **1. Static Analysis & Linting** | `yamllint`, `ansible-lint`, `ansible-playbook --syntax-check` | Seconds | Broken YAML, deprecated module names, unquoted Jinja2, missing FQCNs |
| **2. Schema & Template Unit Tests** | `ansible.utils.validate` (JSON Schema), `pytest` | Seconds | Invalid IP CIDRs, duplicate VLAN IDs, missing required `host_vars` fields |
| **3. Offline Control-Plane Analysis** | **Batfish** (`pybatfish`) | 10–30s | BGP session mismatches, unreachable subnets, shadowed ACL rules, routing loops |
| **4. Live Virtual Lab Integration** | **Containerlab** / **netlab** + Ansible validation playbooks | 1–5 min | Real NOS CLI syntax rejections, actual BGP/OSPF convergence, data-plane pings |

## Validating Data Models with JSON Schema

Before rendering a single config, assert that `host_vars/` obeys your data contract using `ansible.utils.validate`:

```yaml
- name: Validate campus VLAN data model against JSON Schema
  ansible.utils.validate:
    data: "{{ {'vlans': campus_vlans} }}"
    criteria: "{{ lookup('file', 'schemas/vlans.schema.json') | from_json }}"
    engine: ansible.utils.jsonschema
```

If someone accidentally types `vlan_id: 5000` (outside 1–4094) or forgets a VLAN name, the playbook fails immediately in under a second before connecting to any device.

## Running `ansible-lint` with Production Profile

Configure `.ansible-lint` at your repository root:

```yaml
---
profile: production
exclude_paths:
  - .venv/
  - collections/
  - intended/
  - backups/
```

```bash
yamllint .
ansible-lint
ansible-playbook playbooks/site.yml --syntax-check
```

## Integration Testing Against Ephemeral Containerlab

In CI or locally, spin up a clean virtual twin, run your playbook **twice** (first run applies changes; second run asserts `changed=0` for idempotency), run your validation playbook from [Chapter 25 — Troubleshooting and Validation Playbooks](25-troubleshooting-and-validation-playbooks.md), and destroy the lab:

```bash
sudo containerlab deploy -t tests/lab.clab.yml --reconfigure
ansible-playbook -i clab-test/ansible-inventory.yml playbooks/site.yml
ansible-playbook -i clab-test/ansible-inventory.yml playbooks/site.yml --diff | tee /tmp/idempotency.log
ansible-playbook -i clab-test/ansible-inventory.yml playbooks/validate.yml
sudo containerlab destroy -t tests/lab.clab.yml --cleanup
```
