---
title: '08. Containerising & Deploying to Kubernetes'
---

# Containerising & Deploying to Kubernetes

Deploying gRPC to Kubernetes requires attention to HTTP/2 load balancing, native gRPC readiness/liveness probes, automated TLS certificates, and choosing the right ingress mechanism.

## Module Syllabus (6 Lessons)

1. What is Docker & Containerising Go Services
2. What is Kubernetes & What do I Need to do to Deploy There?
3. TLS & Automatic Certificate Renewal Using Let's Encrypt (`cert-manager`)
4. Exposing a gRPC Service Part 1 — via `LoadBalancer` Service
5. Exposing a gRPC Service Part 2 — via `Ingress` (HTTP/2 / gRPC backend)
6. Exposing a gRPC Service Part 3 — via Cloudflare Tunnel (`cloudflared`)

## Three Ways to Expose a gRPC Service

| Exposure Method | How It Works | Trade-offs |
|-----------------|--------------|------------|
| **LoadBalancer Service** | Cloud L4/L7 LB forwards TCP/HTTP2 directly to cluster pods | Simple; L4 LB does not balance individual HTTP/2 streams across pods |
| **Ingress / Gateway API** | L7 proxy (NGINX / Envoy) terminates Let's Encrypt TLS and routes HTTP/2 `backend-protocol: GRPC` | Proper per-RPC L7 load balancing, host/path routing, automated TLS via `cert-manager` |
| **Cloudflare Tunnel** | Outbound `cloudflared` daemon connects cluster to Cloudflare edge with zero open inbound ports | Ideal for homelabs and private clusters; supports gRPC over HTTP/2 |

## Kubernetes Deployment with Native gRPC Probes

Kubernetes 1.27+ supports native `grpc` health probes directly against `grpc.health.v1.Health`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-grpc
spec:
  replicas: 3
  selector:
    matchLabels:
      app: todo-grpc
  template:
    metadata:
      labels:
        app: todo-grpc
    spec:
      containers:
        - name: todo-grpc
          image: ghcr.io/your-org/todo-grpc:v1.0.0
          ports:
            - containerPort: 50051
              name: grpc
          readinessProbe:
            grpc:
              port: 50051
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            grpc:
              port: 50051
            initialDelaySeconds: 10
            periodSeconds: 20
```

## Related Chapters

- Previous: [07. Testing gRPC Services](07-testing-grpc-services.md)
- Next: [09. Using Buf to Manage Protobuf](09-buf-and-connectrpc.md)
- Book foundations: [Docker Multi-Stage Builds](../20-infrastructure/70-docker-multistage.md), [Kubernetes Deployment Basics](../20-infrastructure/75-kubernetes-deploy-basics.md)
