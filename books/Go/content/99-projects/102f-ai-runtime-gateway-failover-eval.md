---
title: '102f. Project: AI Runtime Gateway & Adaptive Load-Shedding Reverse Proxy'
---

# Project: AI Runtime Gateway & Adaptive Load-Shedding Reverse Proxy

Build a production Go reverse proxy that combines **multi-provider LLM automatic failover**, **per-tenant OpenTelemetry token/cost attribution**, **circuit breakers for runaway agent loops**, and **adaptive concurrency / priority-based load shedding** (keeping critical read endpoints alive under 10× spike load).

## Architecture

```text
Clients / AI Agents (X-Tenant-ID, X-Priority: high|low)
        |
        v
[ Go Gateway & Load-Shedding Proxy ]
   ├── Adaptive Concurrency Limiter (Little's Law / Vegas RTT gradient)
   ├── Priority Shedder (sheds low-priority requests first with 503 + Retry-After)
   ├── Per-Tenant Token Budget & Runaway-Loop Circuit Breaker
   ├── Provider Router: Primary LLM (OpenAI/Vertex) --failover--> Secondary (vLLM/Anthropic)
   └── OTel Metrics: llm_tokens_total{tenant,model}, llm_cost_usd_total{tenant}
```

## Adaptive Concurrency & Priority Load-Shedding Middleware

```go
package gateway

import (
	"net/http"
	"sync/atomic"
)

type PriorityLimiter struct {
	inFlight atomic.Int64
	softCap  int64 // shed low-priority traffic above softCap
	hardCap  int64 // shed all traffic above hardCap
}

func NewPriorityLimiter(softCap, hardCap int64) *PriorityLimiter {
	return &PriorityLimiter{softCap: softCap, hardCap: hardCap}
}

func (l *PriorityLimiter) Middleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		cur := l.inFlight.Add(1)
		defer l.inFlight.Add(-1)

		priority := r.Header.Get("X-Priority")
		if cur > l.hardCap || (cur > l.softCap && priority != "high") {
			w.Header().Set("Retry-After", "2")
			http.Error(w, "service overloaded, shedding load", http.StatusServiceUnavailable)
			return
		}
		next.ServeHTTP(w, r)
	})
}
```

## Acceptance Drills

- Simulate a 503 outage on Primary LLM and verify automatic failover to Secondary within < 500ms with zero dropped requests.
- Drive 10× synthetic load with 80% `low` and 20% `high` priority traffic and verify > 99% of `high` priority requests succeed while `low` priority requests shed cleanly with `503`.
- Wire a GitHub Actions eval gate that blocks prompt/model config changes if accuracy or p95 TTFT regresses.
