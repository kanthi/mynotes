---
title: '07. Project: Zero-Trust Tailscale Mesh Onboarding & Policy Automation'
---

# Project: Zero-Trust Tailscale Mesh Onboarding & Policy Automation

Build a **Go CLI + OpenTofu/Terraform wrapper** and customer-ready reference architecture that connects multi-cloud Kubernetes services, a private GitOps server, and developer bastion hosts over a **WireGuard/Tailscale** zero-trust mesh—using ephemeral pre-auth keys, least-privilege ACL tags, subnet routers, and SSH session recording.

## Mesh Topology & Least-Privilege Policy

```text
Developer Laptops (tag:eng)
        |  Tailscale SSH (session recorded, short-lived OIDC)
        v
Private GitOps & K8s API (tag:prod-k8s, subnet router 10.42.0.0/16)
        ^
        |  Ephemeral K8s Pod / Bare-Metal Node (tag:ci-runner, auto-expires on exit)
```

## Declarative Tailscale ACL Policy (`policy.hujson`)

```json
{
  "tagOwners": {
    "tag:prod-k8s": ["autogroup:admin"],
    "tag:ci-runner": ["autogroup:admin"]
  },
  "acls": [
    {
      "action": "accept",
      "src": ["tag:ci-runner"],
      "dst": ["tag:prod-k8s:443,6443"]
    }
  ],
  "ssh": [
    {
      "action": "check",
      "src": ["autogroup:member"],
      "dst": ["tag:prod-k8s"],
      "users": ["ubuntu"]
    }
  ]
}
```

## Acceptance Drills

- Onboard an ephemeral Kubernetes pod with an automated single-use auth key and confirm it can reach `tag:prod-k8s:6443` but is blocked from SSH (`22`) and other subnets.
- Terminate the pod and verify its ephemeral node record is automatically purged from the tailnet.
