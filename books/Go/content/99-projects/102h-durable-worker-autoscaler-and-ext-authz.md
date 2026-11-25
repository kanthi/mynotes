---
title: '102h. Project: Durable Worker Autoscaler & Envoy ext_authz Identity Gateway'
---

# Project: Durable Worker Autoscaler & Envoy `ext_authz` Identity Gateway

Combine two core control-plane Go services:
1. A **Durable Task Worker Autoscaler** that reads per-tenant queue lag, applies stabilization windows and debouncing to prevent replica thrashing, and enforces per-tenant isolation caps.
2. An **Envoy External Authorization (`ext_authz`)** gRPC service in Go that validates OIDC JWTs + Attribute-Based Access Control (ABAC) policies and mTLS peer identities before traffic reaches backend services.

## Debounced Queue-Lag Autoscaler Core

```go
package autoscale

import (
	"math"
	"time"
)

type Policy struct {
	TargetLagPerWorker float64
	MinReplicas        int
	MaxReplicas        int
	ScaleDownCooldown  time.Duration
}

type Controller struct {
	policy        Policy
	current       int
	lastScaleUpAt time.Time
}

func (c *Controller) Decide(now time.Time, queueLag int) int {
	desired := int(math.Ceil(float64(queueLag) / c.policy.TargetLagPerWorker))
	desired = max(c.policy.MinReplicas, min(c.policy.MaxReplicas, desired))

	if desired > c.current {
		c.current = desired
		c.lastScaleUpAt = now
		return c.current
	}
	// Debounce scale-down to prevent oscillation during bursty traffic.
	if desired < c.current && now.Sub(c.lastScaleUpAt) >= c.policy.ScaleDownCooldown {
		c.current = desired
	}
	return c.current
}
```

## Envoy `ext_authz` ABAC Policy Gate

Envoy calls your Go gRPC `AuthorizationServer.Check(ctx, *authv3.CheckRequest)` on every inbound request:
1. Extract `Authorization: Bearer <jwt>` and validate signature, `iss`, `aud`, and `exp`.
2. Extract client SPIFFE/mTLS SAN from `x-forwarded-client-cert` when present.
3. Evaluate ABAC rule `(subject.tenant == resource.tenant && subject.role in allowedRoles)`.
4. Return `codes.OK` (injecting verified `x-auth-subject` and `x-auth-tenant` headers upstream) or `codes.PermissionDenied` (`403`).

## Acceptance Drills

- Replay a sawtooth queue-lag trace and verify scale-up is immediate while scale-down waits for `ScaleDownCooldown`.
- Verify `ext_authz` rejects expired JWTs (`401`), cross-tenant access (`403`), and passes valid requests in < 2ms p99.
