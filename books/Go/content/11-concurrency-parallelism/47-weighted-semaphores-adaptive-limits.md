---
title: Weighted Semaphores, Token-Bucket Rate Limiters, and Adaptive Concurrency Limits
---

# Weighted Semaphores, Token-Bucket Rate Limiters, and Adaptive Concurrency Limits

Microservices rarely fail under steady traffic; they fail from **cascading overload** when downstream latency spikes (e.g., 10 ms -> 500 ms), inflating in-flight requests by 50x and exhausting database pools and RAM. Preventing overload requires three complementary layers: weighted semaphores (`x/sync/semaphore`), continuous-refill token buckets (`x/time/rate`), and latency-driven adaptive concurrency limits.

## Choosing the Right Traffic Control Mechanism

| Mechanism | Governs | Primary Goal | Best Use Case |
|---|---|---|---|
| `semaphore.Weighted` | Total concurrent units (FIFO) | Memory & resource isolation | Bounding DB connections or heavy payload memory budgets |
| `rate.Limiter` (Token Bucket) | Requests/sec + burst (no ticker goroutine) | Burst smoothing & API fairness | Public HTTP endpoints, per-tenant API throttling |
| Adaptive Gradient Limiter | Dynamic in-flight slots via RTT ratio | Preventing cascading collapse | Inter-service RPCs to shared downstream databases |

- **FIFO starvation prevention in `semaphore.Weighted`:** If a 10-permit request is queued, smaller 1-permit arrivals queue behind it rather than cutting ahead.
- **Zero-ticker math in `rate.Limiter`:** Computes `min(burst, tokens + (now - last) * limit)` on each call, supporting 100,000+ per-key limiters with zero background goroutines.

## Adaptive Gradient Concurrency Limiter in Go 1.27

Inspired by TCP Vegas congestion control, the gradient limiter compares baseline `minRTT` against observed `sampleRTT`:

```text
gradient = clamp(minRTT / sampleRTT, 0.5, 1.0)
newLimit = clamp(currentLimit * gradient + headroom, minLimit, maxLimit)
```

```go
package concurrency

import (
	"errors"
	"net/http"
	"sync"
	"sync/atomic"
	"time"
)

var ErrOverloaded = errors.New("limiter: system overloaded, request dropped")

type AdaptiveLimiter struct {
	mu           sync.RWMutex
	currentLimit float64
	minLimit     float64
	maxLimit     float64
	minRTT       time.Duration
	inflight     atomic.Int64
}

func NewAdaptiveLimiter(initialLimit, minLimit, maxLimit float64) *AdaptiveLimiter {
	return &AdaptiveLimiter{
		currentLimit: initialLimit,
		minLimit:     minLimit,
		maxLimit:     maxLimit,
		minRTT:       time.Hour,
	}
}

func (l *AdaptiveLimiter) Acquire() (func(), error) {
	l.mu.RLock()
	limit := l.currentLimit
	l.mu.RUnlock()

	if float64(l.inflight.Add(1)) > limit {
		l.inflight.Add(-1)
		return nil, ErrOverloaded
	}
	start := time.Now()
	return func() {
		rtt := time.Since(start)
		l.inflight.Add(-1)
		l.record(rtt)
	}, nil
}

func (l *AdaptiveLimiter) record(rtt time.Duration) {
	l.mu.Lock()
	defer l.mu.Unlock()
	if rtt < l.minRTT {
		l.minRTT = rtt
	}
	gradient := min(1.0, max(0.5, float64(l.minRTT)/float64(rtt)))
	l.currentLimit = min(l.maxLimit, max(l.minLimit, l.currentLimit*gradient+2.0))
}

func AdaptiveLimitMiddleware(limiter *AdaptiveLimiter, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		release, err := limiter.Acquire()
		if err != nil {
			http.Error(w, "503 Service Overloaded", http.StatusServiceUnavailable)
			return
		}
		defer release()
		next.ServeHTTP(w, r)
	})
}
```

Test token-bucket refills and adaptive backoff deterministically in nanoseconds using `testing/synctest` (`synctest.Test` + `synctest.Sleep`).

## Related Chapters

- [258 Semaphores and Rendezvous](../10-concurrency-ground-up/258-semaphores-rendezvous.md)
- [282 Rate Limiting](../15-web-development-in-go/282-rate-limiting.md)
- [04 Resilience, Backpressure, and Load Shedding](../14-network-performance/04-resilience-backpressure-and-load-shedding.md)
