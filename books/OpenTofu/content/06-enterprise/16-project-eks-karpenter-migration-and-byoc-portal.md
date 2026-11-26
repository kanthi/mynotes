---
title: '16. Project: Zero-Downtime ECS-to-EKS Migration, Karpenter & BYOC Portal'
---

# Project: Zero-Downtime ECS-to-EKS Migration, Karpenter & BYOC Portal

Write modular **OpenTofu** configurations to stand up a production EKS cluster with **Karpenter**, **HPA/VPA**, **OIDC Workload Identity**, and **Vault External Secrets**; simulate weighted traffic shifting from ECS/Docker Compose to Kubernetes with zero downtime; and package a **"Day-0 to Day-2" Self-Service BYOC Portal** (OpenTofu + Helm + GitHub Actions `workflow_dispatch`) for isolated customer VPC provisioning and clean teardown.

## Weighted Traffic Shift Strategy

```text
DNS / ALB Weighted Target Group
   ├── 90% -> Legacy ECS / Compose Service  --->  50%  --->  0% (decommission)
   └── 10% -> EKS Service (Karpenter Spot/On-Demand + HPA) ---> 50% ---> 100%
```

## Karpenter `NodePool` Consolidation Snippet

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: general-purpose
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64", "arm64"]
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 60s
```

## Acceptance Drills

- Shift weights `100/0 -> 90/10 -> 50/50 -> 0/100` under continuous synthetic load and verify 0% HTTP 5xx errors.
- Scale workload down by 70% and verify Karpenter consolidates underutilized nodes and exports cost-savings metrics to Grafana.
- Trigger the BYOC `workflow_dispatch` to spin up an isolated environment in a fresh VPC and verify `tofu destroy` leaves zero orphaned resources.
