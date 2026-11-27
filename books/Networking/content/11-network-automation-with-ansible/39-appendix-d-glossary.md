# Appendix D — Glossary

| Term | Definition |
| :--- | :--- |
| **AAP (Ansible Automation Platform)** | Red Hat's enterprise automation suite combining Automation Controller (upstream AWX), Automation Hub, Event-Driven Ansible, and Execution Environments. |
| **AWX** | Open-source upstream web UI, REST API, RBAC, and job-scheduling engine for Ansible. |
| **Check Mode (`--check`)** | Dry-run execution flag instructing modules to report what *would* change without applying modifications to target devices. |
| **Collection** | Standard packaging format (`namespace.collection`, e.g., `cisco.ios`) distributing Ansible modules, plugins, and roles. |
| **Control Node** | The Linux/macOS host (or Execution Environment container) running `ansible-playbook` and initiating network connections. |
| **Execution Environment (EE)** | Immutable OCI container image bundling `ansible-core`, Python dependencies, OS libraries, and pinned Ansible collections built with `ansible-builder`. |
| **Facts** | Structured key/value metadata automatically gathered from devices (model, OS version, serial number, interfaces, routing state). |
| **FQCN (Fully Qualified Collection Name)** | Explicit three-part module/plugin identifier (`namespace.collection.module`, e.g., `cisco.ios.ios_vlans`). |
| **Handler** | A special task triggered via `notify:` only when a task reports `changed`, executed once at the end of a play. |
| **`httpapi`** | Persistent connection plugin communicating with network device REST/JSON-RPC APIs over HTTPS (FortiOS, Arista eAPI, NX-API). |
| **Idempotency** | Property guaranteeing that applying the same playbook repeatedly yields the exact same desired state with zero unintended side effects after the first run. |
| **Inventory** | Static YAML file or dynamic plugin output defining managed hosts, group hierarchies, and host/group variables. |
| **Jinja2** | Python templating engine used by Ansible to render dynamic configuration files and transform variables via filters. |
| **`netconf`** | Persistent connection plugin communicating over NETCONF/SSH (TCP 830) with structured XML/YANG datastores (Junos, IOS-XR, IOS-XE). |
| **`network_cli`** | Persistent connection plugin managing interactive SSH CLI sessions using vendor terminal and cliconf plugins. |
| **Resource Module** | Schema-driven network module supporting `merged`, `replaced`, `overridden`, `deleted`, `gathered`, `rendered`, and `parsed` states. |
| **Source of Truth (SoT)** | Authoritative data store (such as NetBox, Nautobot, or Git-tracked YAML) defining the *intended* state of the network. |
| **Vault (`ansible-vault`)** | Built-in AES-256 encryption tool for protecting passwords, API tokens, and private keys stored in Git repositories. |
