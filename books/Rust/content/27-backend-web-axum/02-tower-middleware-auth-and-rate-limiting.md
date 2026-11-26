---
title: Tower Middleware, Authentication, and Rate Limiting
---

# Tower Middleware, Authentication, and Rate Limiting

In Axum, middleware is not a framework-specific callback—it is a composable **`tower::Layer`** wrapping a **`tower::Service`**. That design lets you stack tracing, request timeouts, body size caps, CORS, bearer authentication, and concurrency limits into one predictable pipeline with zero dynamic allocation per layer.

## Ordering Matters: `ServiceBuilder` Top-to-Bottom

Calling `.layer(...)` repeatedly on an Axum `Router` wraps layers inside-out (bottom-to-top), which is easy to misread. Wrapping layers inside **`tower::ServiceBuilder`** executes them cleanly **top-to-bottom** on incoming requests:

```rust
use axum::{
    extract::Request,
    http::{header, StatusCode},
    middleware::{self, Next},
    response::Response,
    routing::get,
    Router,
};
use std::time::Duration;
use tower::ServiceBuilder;
use tower_http::{
    limit::RequestBodyLimitLayer,
    timeout::TimeoutLayer,
    trace::TraceLayer,
};

async fn require_bearer(req: Request, next: Next) -> Result<Response, StatusCode> {
    let auth = req
        .headers()
        .get(header::AUTHORIZATION)
        .and_then(|v| v.to_str().ok());

    match auth {
        Some(h) if h.starts_with("Bearer ") && h.len() > 16 => Ok(next.run(req).await),
        _ => Err(StatusCode::UNAUTHORIZED),
    }
}

pub fn hardened_router() -> Router {
    let protected = Router::new()
        .route("/v1/orders", get(|| async { "orders ok" }))
        .route_layer(middleware::from_fn(require_bearer));

    Router::new()
        .route("/healthz", get(|| async { "ok" }))
        .merge(protected)
        .layer(
            ServiceBuilder::new()
                .layer(TraceLayer::new_for_http())
                .layer(TimeoutLayer::new(Duration::from_secs(10)))
                .layer(RequestBodyLimitLayer::new(1024 * 1024)), // 1 MiB body cap
        )
}
```

> [!IMPORTANT]
> Always apply `RequestBodyLimitLayer` and `TimeoutLayer` globally on public-facing APIs. Without a body cap, a single client streaming an unbounded JSON body can exhaust server memory before your handler even executes.

## Token-Bucket Rate Limiting in Pure Rust

For per-IP or per-API-key rate limiting (often backed by `governor` or Redis Lua scripts in production), the core algorithm is a **token bucket** that refills smoothly over elapsed time:

```rust
use std::time::Instant;

pub struct TokenBucket {
    capacity: f64,
    tokens: f64,
    refill_per_sec: f64,
    last_refill: Instant,
}

impl TokenBucket {
    pub fn new(capacity: f64, refill_per_sec: f64) -> Self {
        Self { capacity, tokens: capacity, refill_per_sec, last_refill: Instant::now() }
    }

    pub fn allow(&mut self) -> bool {
        let now = Instant::now();
        let elapsed = now.duration_since(self.last_refill).as_secs_f64();
        self.tokens = (self.tokens + elapsed * self.refill_per_sec).min(self.capacity);
        self.last_refill = now;
        if self.tokens >= 1.0 {
            self.tokens -= 1.0;
            true
        } else {
            false
        }
    }
}
```

## Hands-On Lab: Verify Burst and Throttling Behavior

Compile and run the `TokenBucket` struct locally to confirm it allows an initial burst of 3 requests and rejects the 4th immediately:

```rust
fn main() {
    let mut bucket = TokenBucket::new(3.0, 1.0);
    let results: Vec<bool> = (0..5).map(|_| bucket.allow()).collect();
    println!("Immediate burst of 5: {results:?}");
    assert_eq!(results, vec![true, true, true, false, false]);
}
```
