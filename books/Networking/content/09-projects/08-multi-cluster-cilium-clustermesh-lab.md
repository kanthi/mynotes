---
title: '08. Project: Multi-Cluster Kubernetes Fleet Simulator with Cilium ClusterMesh'
---

# Project: Multi-Cluster Kubernetes Fleet Simulator with Cilium ClusterMesh

Spin up 2–3 local `kind` / `k3s` clusters connected via **Cilium ClusterMesh** to practice cross-cluster service discovery, **CiliumNetworkPolicy** zero-trust enforcement, **Hubble eBPF** live flow observability, and federated ingress failover.

## Topology

```text
Cluster A (10.1.0.0/16)  <=== Cilium ClusterMesh (WireGuard / VXLAN) ===>  Cluster B (10.2.0.0/16)
  ├── Global Service: api.prod (service.cilium.io/global: "true")
  ├── CiliumNetworkPolicy (L3/L4/L7 identity-based allowlist)
  └── Hubble UI / CLI (live DNS, HTTP, drop-reason eBPF flow telemetry)
```

## Global Service Annotation & Failover Drill

Annotate the `Service` in both clusters with `service.cilium.io/global: "true"` and `service.cilium.io/affinity: "local"` so traffic stays local when healthy and automatically fails over across the ClusterMesh tunnel when local endpoints go down.

## Acceptance Drills

- Scale `api` replicas in Cluster A to `0` and verify requests from Cluster A transparently route to Cluster B via ClusterMesh.
- Apply a `CiliumNetworkPolicy` denying egress from `untrusted` pods and observe real-time `DROPPED` verdicts in `hubble observe --verdict DROPPED`.
