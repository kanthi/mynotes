---
title: '06. Client Service Config & Load Balancing'
---

# Client Service Config & Load Balancing

Because gRPC multiplexes all RPCs over a single persistent HTTP/2 TCP connection, naive Layer-4 load balancers pin all traffic from a client to one backend pod. **Client-side load balancing** and **gRPC Service Config** solve both traffic distribution and automatic retries.

## Module Syllabus (5 Lessons)

1. Client Service Config & Timeouts
2. Automatic Client Retries
3. Introduction to Client-Side Load Balancing
4. Round Robin Load Balancing
5. Creating a Custom Load Balancing Policy

## Declarative Service Config (Round-Robin + Retries + Timeouts)

Pass a JSON service config via `grpc.WithDefaultServiceConfig(...)` when creating the client. Only mark **idempotent** RPCs (or safe transient status codes like `UNAVAILABLE`) as retryable.

```go
package client

import (
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials"
)

const serviceConfigJSON = `{
	"loadBalancingConfig": [{"round_robin": {}}],
	"methodConfig": [{
		"name": [{"service": "todo.v1.TodoService"}],
		"timeout": "2s",
		"retryPolicy": {
			"maxAttempts": 4,
			"initialBackoff": "0.1s",
			"maxBackoff": "1s",
			"backoffMultiplier": 2.0,
			"retryableStatusCodes": ["UNAVAILABLE"]
		}
	}]
}`

func DialBalanced(target string, creds credentials.TransportCredentials) (*grpc.ClientConn, error) {
	// Use dns:///my-headless-svc:50051 so gRPC resolves all backend pod IPs for round_robin.
	return grpc.NewClient(
		target,
		grpc.WithTransportCredentials(creds),
		grpc.WithDefaultServiceConfig(serviceConfigJSON),
	)
}
```

## Custom Load Balancing Policies

When `round_robin` is not enough (for example, weighted routing, zone-aware affinity, or least-in-flight selection), implement `balancer.Builder` and `balancer.Picker` (using `base.NewBalancerBuilder`) and register the policy at init time with `balancer.Register(...)`.

## Related Chapters

- Previous: [05. Interceptors, Metadata & Authorization](05-interceptors-metadata-authz.md)
- Next: [07. Testing gRPC Services](07-testing-grpc-services.md)
- Book foundations: [Retry and Circuit Breaker Patterns](../18-distributed-infra/152-retry-circuit-breaker-patterns.md)
