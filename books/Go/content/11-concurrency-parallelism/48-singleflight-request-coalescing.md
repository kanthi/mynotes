---
title: Singleflight, Request Coalescing, and Cache Stampede Defense
---

# Singleflight, Request Coalescing, and Cache Stampede Defense

When a hot cache key expires under 10,000 req/s, all 10,000 goroutines miss simultaneously and hammer the origin database (**Cache Stampede / Thundering Herd**). `golang.org/x/sync/singleflight` deduplicates concurrent calls for the same key so only **1** goroutine executes the origin fetch while the remaining 9,999 wait and share its result.

```text
10,000 Concurrent Requests (Hot Key Expires -> Cache Miss)
              |
              +-----------------------------------+
              | (Without Singleflight)            | (With Singleflight)
              v                                   v
+-------------------------------+   +-------------------------------+
| 10,000 Simultaneous DB Queries|   | 1 Single Execution to DB      |
| - Pool exhausted, DB outage   |   | 9,999 goroutines share result |
+-------------------------------+   +-------------------------------+
```

## The Shared Context Cancellation Trap and `DoChan` Fix

> [!CAUTION]
> Passing Caller 1's request `ctx` into `group.Do(key, func() { return fetchDB(ctx, key) })` is a classic production bug: if Caller 1 disconnects after 50 ms, `ctx` cancels the shared DB query and fails all other co-waiting callers with `context.Canceled`.

Always run the origin fetch with a detached timeout context via `group.DoChan` and let each caller `select` against its own `ctx.Done()`:

```go
package coalescer

import (
	"context"
	"time"

	"golang.org/x/sync/singleflight"
)

// TypedGroup provides type-safe request coalescing without caller context leakage.
type TypedGroup[T any] struct {
	group singleflight.Group
}

func (g *TypedGroup[T]) DoWithTimeout(
	ctx context.Context,
	key string,
	originTimeout time.Duration,
	fn func(context.Context) (T, error),
) (T, error) {
	ch := g.group.DoChan(key, func() (any, error) {
		bgCtx, cancel := context.WithTimeout(context.Background(), originTimeout)
		defer cancel()
		return fn(bgCtx)
	})

	select {
	case <-ctx.Done():
		var zero T
		return zero, ctx.Err()
	case res := <-ch:
		if res.Err != nil {
			var zero T
			return zero, res.Err
		}
		return res.Val.(T), nil
	}
}

// Forget evicts an in-flight key immediately (e.g., after a write invalidation).
func (g *TypedGroup[T]) Forget(key string) { g.group.Forget(key) }
```

- **Panic safety:** `singleflight` recovers panics in `fn()`, marks the call finished so waiters never deadlock, and re-panics on all waiting callers.
- **Combine with probabilistic early refresh (XFetch)** and test coalescing deterministically with `testing/synctest` (`synctest.Test` + `synctest.Sleep`).

## Related Chapters

- [154 Caching Patterns](../18-distributed-infra/154-caching-patterns.md)
- [223 testing/synctest Bubbles](../25-go-deep-dives/223-testing-synctest-bubbles.md)
