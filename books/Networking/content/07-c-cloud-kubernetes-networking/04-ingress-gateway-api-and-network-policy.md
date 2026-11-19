---
title: Ingress, Gateway API, and NetworkPolicy
---

# Ingress, Gateway API, and NetworkPolicy

Kubernetes Services handle L4 load balancing, but production applications need L7 HTTP/gRPC routing, TLS termination, traffic splitting, and strict east-west micro-segmentation. This chapter compares the legacy `Ingress` API with the modern **Kubernetes Gateway API**, explains how `NetworkPolicy` and Cilium L7/FQDN policies lock down Pod traffic, and establishes a repeatable packet-level debugging workflow.

## Learning goals

By the end of this chapter you can:

- Explain the structural limitations of `Ingress` and how `GatewayClass`, `Gateway`, and `HTTPRoute` solve them
- Configure TLS termination, header-based routing, and weighted canary splits with Gateway API
- Write default-deny `NetworkPolicy` baselines without breaking CoreDNS or health probes
- Distinguish Kubernetes L3/L4 `NetworkPolicy` from Cilium L7/FQDN policies
- Debug live Pod traffic using `kubectl debug`, `crictl`, `nsenter`, and `hubble observe`

## Ingress limitations vs Kubernetes Gateway API

The original `networking.k8s.io/v1` `Ingress` resource was designed for simple host/path HTTP routing. As soon as teams needed header matching, weighted canary splits, request mirroring, gRPC routing, or cross-namespace delegation, every controller (NGINX, Traefik, Envoy, ALB) invented incompatible **controller-specific annotations**:

```yaml
# Legacy Ingress: non-portable vendor annotations for basic L7 features
metadata:
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
```

**Kubernetes Gateway API** replaces this monolith with a **role-oriented, portable resource model**:

| Resource | Owned by | Responsibility |
|----------|----------|----------------|
| **`GatewayClass`** | Infrastructure provider | Defines which controller implements the gateway (e.g., `cilium`, `envoy-gateway`, `istio`, `gke-l7-global`) |
| **`Gateway`** | Platform / NetOps team | Instantiates a load balancer listener: ports, protocols (HTTP/HTTPS/TCP/TLS/gRPC), TLS certificates, and which namespaces may attach routes |
| **`HTTPRoute` / `GRPCRoute` / `TCPRoute`** | Application team | Declares hostnames, path/header/query matches, traffic weights, filters, and backend Services |
| **`ReferenceGrant`** | Target namespace owner | Explicitly authorizes a route in Namespace A to forward traffic to a Service or Secret in Namespace B |

### Gateway API example: TLS termination and 90/10 canary split

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: prod-edge
  namespace: infra-gateway
spec:
  gatewayClassName: cilium
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: "api.example.com"
      tls:
        mode: Terminate
        certificateRefs:
          - name: api-example-tls
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              gateway-access: "true"
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: payments-route
  namespace: payments
spec:
  parentRefs:
    - name: prod-edge
      namespace: infra-gateway
  hostnames:
    - "api.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /v2/charge
          headers:
            - name: X-Canary
              value: "true"
      backendRefs:
        - name: payments-v2
          port: 8080
    - matches:
        - path:
            type: PathPrefix
            value: /v2/charge
      backendRefs:
        - name: payments-v1
          port: 8080
          weight: 90
        - name: payments-v2
          port: 8080
          weight: 10
```

> [!TIP]
> **TLS Passthrough vs Terminate:** Use `tls.mode: Terminate` when the Gateway needs to inspect HTTP paths, headers, or inject `X-Forwarded-For`. Use `tls.mode: Passthrough` with a `TLSRoute` when the backend application must perform end-to-end TLS termination (for example, client-certificate mTLS directly to an internal vault or database).

## Kubernetes `NetworkPolicy` fundamentals

By default, Kubernetes networking is **flat and open**: every Pod in every namespace can connect to every other Pod. A `NetworkPolicy` is enforced by the CNI plugin (Calico, Cilium, Antrea, etc.—note that basic CNIs like Flannel ignore `NetworkPolicy` silently).

### Key behavioral rules

1. **Additive allow-lists**: Once *any* `NetworkPolicy` selects a Pod for `Ingress` (or `Egress`), that direction flips to **default deny**, and only traffic matching at least one `allow` rule is permitted.
2. **Stateful connection tracking**: Allowing ingress TCP 8080 automatically permits the return packets of that established TCP connection; you do not need a matching egress rule for replies.
3. **Both sides must agree**: For Pod A to reach Pod B, Pod A's egress policies *and* Pod B's ingress policies must both permit the flow.
4. **Post-DNAT evaluation**: Standard `NetworkPolicy` rules evaluate **Pod IPs**, not Service `ClusterIP`s. Allow the destination Pod's labels and target container port (e.g., `8080`), not the Service port (`80`).

### Default-deny baseline and the DNS egress trap

When applying a default-deny egress policy to a namespace, **always explicitly allow CoreDNS** (UDP and TCP port 53 to `kube-system`), otherwise every outbound connection stalls on DNS resolution:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: payments
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: payments
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

> [!WARNING]
> **YAML list trap (`AND` vs `OR`):** In the `allow-dns-egress` manifest above, `namespaceSelector` and `podSelector` are under a **single** list item (`-`), meaning they are ANDed together (only `kube-dns` Pods inside `kube-system`). Putting a second `-` before `podSelector` turns it into an OR, accidentally allowing egress to *every* Pod in `kube-system`!

## Cilium L7 and FQDN policies

Standard Kubernetes `NetworkPolicy` operates only at L3/L4 (IPs, CIDRs, labels, ports). It cannot filter by HTTP method/path or allow egress to dynamic external SaaS domains like `api.stripe.com` whose CDN IP addresses rotate constantly.

`CiliumNetworkPolicy` extends enforcement with:

- **DNS-aware FQDN egress (`toFQDNs`)**: Cilium's transparent DNS proxy intercepts the Pod's DNS response, caches the resolved IP-to-hostname mapping with its TTL, and dynamically programs BPF policy maps for those IPs.
- **L7 protocol rules**: Enforces HTTP method/path, gRPC service/method, or Kafka topic allow-lists via an inline Envoy proxy managed by the Cilium agent.

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: payments-stripe-egress
  namespace: payments
spec:
  endpointSelector:
    matchLabels:
      app: payments-api
  egress:
    - toEndpoints:
        - matchLabels:
            io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: ANY
          rules:
            dns:
              - matchPattern: "*.stripe.com"
    - toFQDNs:
        - matchName: "api.stripe.com"
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
```

## Practical Kubernetes network debugging workflow

Modern container images are frequently `distroless` or `scratch`, containing no `curl`, `dig`, `ip`, or `tcpdump`. Use this four-step workflow to diagnose packet loss without mutating production images:

### 1. Ephemeral debug containers (`kubectl debug`)

Attach a toolbox container (`nicolaka/netshoot`) directly into the target Pod's network namespace:

```bash
kubectl debug -it payments-api-7d9c8f4b2-xk9lm -n payments \
  --image=nicolaka/netshoot \
  --target=payments-api \
  -- sh

# Inside netshoot (shares exact Pod eth0, routes, and /etc/resolv.conf):
ip -br addr
ip route
dig +stats api.stripe.com
curl -sv --connect-timeout 3 https://api.stripe.com/health
```

### 2. Node-level namespace entry (`crictl` + `nsenter`)

When the API server is unreachable or ephemeral containers are blocked by admission policy, SSH to the worker node and enter the Pod's network namespace directly:

```bash
# Find the container ID and host PID via crictl
CID=$(sudo crictl ps --name payments-api -q | head -n1)
PID=$(sudo crictl inspect --output go-template --template '{{.info.pid}}' "$CID")

# Run host-installed ss, ip, or tcpdump inside the Pod's network namespace
sudo nsenter -t "$PID" -n ip -br addr
sudo nsenter -t "$PID" -n ss -tanp
sudo nsenter -t "$PID" -n tcpdump -ni eth0 -c 20 'tcp port 443 or udp port 53'
```

### 3. Observing policy drops with `hubble observe` (Cilium)

When using Cilium, **Hubble** streams real-time eBPF flow verdicts directly from the kernel, pinpointing exact `NetworkPolicy` drops and DNS responses without running `tcpdump`:

```bash
# Watch dropped flows in the payments namespace in real time
hubble observe --namespace payments --verdict DROPPED -f

# Trace HTTP and DNS flows for a specific Pod
hubble observe --pod payments/payments-api-7d9c8f4b2-xk9lm --protocol dns,http --last 50
```

### 4. Isolating the failing layer

| Symptom | Most likely culprit | First command to run |
|---------|---------------------|----------------------|
| `NXDOMAIN` or 5-second DNS stall | `ndots:5`, missing DNS egress policy, or CoreDNS overload | `dig @10.96.0.10 kubernetes.default.svc.cluster.local` |
| `SYN` sent, zero `SYN-ACK`, `DROPPED` in Hubble | `NetworkPolicy` ingress/egress mismatch | `hubble observe --verdict DROPPED` |
| `Connection refused` (`RST`) | App listening on `127.0.0.1` instead of `0.0.0.0`, or wrong `targetPort` | `nsenter -t $PID -n ss -tlnp` |
| Works Pod-to-Pod, fails via `ClusterIP` | Empty `EndpointSlice` (label mismatch or failing readiness probe) | `kubectl get endpointslices -l kubernetes.io/service-name=<svc>` |
| Works internally, fails via Gateway | Unattached `HTTPRoute`, missing `ReferenceGrant`, or TLS secret mismatch | `kubectl describe httproute <name>` |

## Summary

Gateway API separates cluster infrastructure (`GatewayClass`, `Gateway`) from application routing (`HTTPRoute`), eliminating vendor-specific Ingress annotations. Combining default-deny `NetworkPolicy` baselines with explicit CoreDNS rules—and Cilium FQDN/L7 policies where needed—locks down east-west and egress traffic, while `kubectl debug`, `nsenter`, and `hubble observe` give you surgical visibility into any failing flow. In the next part, we drop down into the Linux kernel dataplane itself, starting with [Linux eBPF, XDP, and TC Programmable Dataplanes](../07-d-dataplanes-and-traffic-delivery/01-ebpf-xdp-and-tc-dataplane.md).
