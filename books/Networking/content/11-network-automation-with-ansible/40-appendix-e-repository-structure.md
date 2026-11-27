# Appendix E — Repository Structure

## Production Network Automation Repository Blueprint

Organize your Git repository so any engineer can locate inventories, data models, reusable roles, templates, schemas, tests, and CI workflows in seconds:

```text
net-automation/
├── .ansible-lint                    # Linting profile and rule exclusions
├── .gitignore                       # Ignores .venv, .vault_pass, triage/, clab-*/
├── .pre-commit-config.yaml          # Local pre-commit hooks (yamllint, ansible-lint, secrets)
├── .yamllint.yml                    # YAML formatting standards
├── .github/
│   └── workflows/
│       └── netops-ci.yml            # Lint, render, lab test, and gated prod deploy
├── Makefile                         # One-word shortcuts (make lint, make build, make check)
├── README.md                        # Onboarding, runbooks, and emergency rollback steps
├── ansible.cfg                      # Project-local Ansible defaults and timeouts
├── execution-environment.yml        # ansible-builder definition for AWX/AAP EE
├── requirements.txt                 # Pinned Python dependencies (netaddr, textfsm, etc.)
├── collections/
│   └── requirements.yml             # Pinned vendor collections (cisco.ios, junos, fortios)
├── inventory/
│   ├── hosts.yml                    # Static lab/bootstrap inventory
│   ├── netbox.yml                   # Dynamic NetBox Source of Truth inventory plugin
│   ├── group_vars/
│   │   ├── all/
│   │   │   ├── standards.yml        # Global NTP, DNS, syslog, banners
│   │   │   └── vault.yml            # AES-256 encrypted credentials (ansible-vault)
│   │   ├── ios.yml                  # Cisco IOS-XE connection and platform vars
│   │   ├── junos.yml                # Juniper Junos NETCONF vars
│   │   ├── fortios.yml              # Fortinet FortiOS HTTPAPI vars
│   │   ├── campus_access.yml        # Campus VLANs and 802.1X profiles
│   │   └── site_london.yml          # Site-specific prefixes and local servers
│   └── host_vars/
│       ├── rtr-edge-01.yml          # Device-specific loopbacks, BGP peers, interfaces
│       └── sw-bldg1-fl2.yml         # Switch-specific port profile mappings
├── playbooks/
│   ├── site.yml                     # Master end-to-end orchestration playbook
│   ├── build_configs.yml            # Offline Jinja2 config rendering -> intended/configs/
│   ├── deploy_configs.yml           # Batched deployment with pre/post validation
│   ├── backup.yml                   # Daily Git-backed config snapshots
│   ├── compliance.yml               # Nightly golden-config drift audit
│   ├── upgrade_prestage.yml         # Image copy and MD5 checksum verification
│   └── validate.yml                 # Pre/post BGP, OSPF, and ping assertions
├── roles/
│   ├── base_mgmt/                   # Multi-vendor NTP, syslog, SNMPv3, banners
│   ├── aaa_tacacs/                  # AAA and privilege hardening
│   ├── campus_access/               # VLANs, trunks, and staged 802.1X
│   └── perimeter_security/          # Cross-platform ACLs and firewall policies
├── schemas/
│   ├── vlans.schema.json            # JSON Schema validating campus_vlans data model
│   └── bgp.schema.json              # JSON Schema validating BGP peer structures
├── templates/
│   ├── ios/                         # Modular Cisco IOS-XE Jinja2 partials
│   ├── junos/                       # Modular Juniper Junos Jinja2 partials
│   ├── reports/                     # CSV and Markdown audit report templates
│   └── ttp/                         # Custom TTP CLI parsing templates
├── intended/
│   └── configs/                     # Generated per-device configs reviewed in Git PRs
├── backups/                         # Automated daily running-config snapshots
└── tests/
    └── lab.clab.yml                 # Ephemeral Containerlab topology for CI integration tests
```

## Convenience `Makefile`

Make the safe path the easiest path for every engineer on the team:

```makefile
.PHONY: lint build check validate

lint:
	yamllint .
	ansible-lint
	ansible-playbook playbooks/site.yml --syntax-check

build:
	ansible-playbook playbooks/build_configs.yml

check:
	ansible-playbook playbooks/deploy_configs.yml --check --diff

validate:
	ansible-playbook playbooks/validate.yml
```
