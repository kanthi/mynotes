# Chapter 29 — Version Control and Workflow

## Git as the Backbone of Network Operations

When multiple engineers automate the same network, ad-hoc playbooks sitting on individual laptops cause conflicting changes and lost work. Treat your network automation repository with the same discipline as production application code: **protected `main` branch, short-lived feature branches, mandatory pull request reviews, and linear commit history**.

## Trunk-Based Workflow for NetOps Teams

```text
main (protected: always matches production intended state)
  ├── feat/chg-4210-add-vlan-45-bldg2   --> PR --> CI lint + lab test --> 1 peer approval --> Merge
  └── fix/inc-8812-bgp-prefix-filter    --> PR --> CI lint + check    --> Emergency approval --> Merge
```

1. **Branch from `main`** with a descriptive prefix (`feat/`, `fix/`, `chore/`) and ticket reference.
2. **Update data or templates** and run `build_configs.yml` so the rendered `intended/configs/` diff is visible right inside the Pull Request.
3. **Open a Pull Request** that triggers automated `yamllint`, `ansible-lint`, and `--syntax-check`.
4. **Peer review:** Another network engineer inspects the rendered CLI diff—catching a wrong subnet mask or missing `add` keyword before any router is touched.
5. **Merge and deploy:** Merging to `main` triggers (or gates) the AWX/CI deployment job.

## Essential `.gitignore` for Network Automation Repos

Never commit transient state, plaintext vault passwords, or local virtual environments:

```text
# .gitignore
.venv/
__pycache__/
*.retry
.vault_pass
*.key
*.pem
clab-*/
triage/
.ansible/
```

## Pre-Commit Hooks: Catch Mistakes Before `git push`

Install `pre-commit` with `.pre-commit-config.yaml` so broken YAML or accidental plaintext secrets are blocked locally before they ever leave an engineer's workstation:

```yaml
repos:
  - repo: https://github.com/adrienverge/yamllint
    rev: v1.35.1
    hooks:
      - id: yamllint
  - repo: https://github.com/ansible/ansible-lint
    rev: v24.9.0
    hooks:
      - id: ansible-lint
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.5.0
    hooks:
      - id: detect-secrets
```
