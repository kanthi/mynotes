---
title: Reverse Proxies and Load Balancers in Rust
---

# Reverse Proxies and Load Balancers in Rust

Rust has rapidly become the language of choice for edge proxies and service meshes—powering Cloudflare's **Pingora**, Linkerd's **linkerd2-proxy**, and **Sozu**—because predictable p99 latency without GC stop-the-world pauses directly improves tail latency across every hop in a microservice call graph.

## Core Mechanics of an L4/L7 Proxy

1. **Zero-copy bidirectional forwarding (`tokio::io::copy_bidirectional`):** Copies bytes between client and upstream sockets using pooled buffers while respecting TCP half-close (`shutdown()`).
2. **Upstream selection & active health checks:** Tracks backend pool health in an `Arc<RwLock<Vec<Backend>>>` or lock-free `ArcSwap`, routing around unhealthy endpoints via round-robin, least-connections, or power-of-two-choices (P2C).
3. **Connection pooling & keep-alive:** Reusing upstream TLS connections avoids repeating 2-RTT handshakes on every HTTP request.
4. **Timeouts on every phase:** Set distinct connect timeouts (e.g., 500 ms), header-read timeouts (5 s), and idle keep-alive timeouts (60 s) so slowloris clients cannot exhaust file descriptors.

## Async L4 TCP Load Balancer with Round-Robin and Failover

```rust
use std::io;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;
use std::time::Duration;
use tokio::io::copy_bidirectional;
use tokio::net::{TcpListener, TcpStream};
use tokio::time::timeout;

struct Pool {
    backends: Vec<String>,
    cursor: AtomicUsize,
}

impl Pool {
    fn next_backend(&self) -> &str {
        let idx = self.cursor.fetch_add(1, Ordering::Relaxed) % self.backends.len();
        &self.backends[idx]
    }
}

async fn proxy_conn(mut client: TcpStream, pool: Arc<Pool>) -> io::Result<()> {
    let target = pool.next_backend().to_string();
    let mut upstream = timeout(Duration::from_millis(500), TcpStream::connect(&target))
        .await
        .map_err(|_| io::Error::new(io::ErrorKind::TimedOut, "upstream connect timeout"))??;

    let (tx, rx) = copy_bidirectional(&mut client, &mut upstream).await?;
    eprintln!("proxied {target}: client->up={tx}B, up->client={rx}B");
    Ok(())
}

pub async fn run_lb(listen_addr: &str, backends: Vec<String>) -> io::Result<()> {
    let listener = TcpListener::bind(listen_addr).await?;
    let pool = Arc::new(Pool { backends, cursor: AtomicUsize::new(0) });

    loop {
        let (client, _) = listener.accept().await?;
        let pool = Arc::clone(&pool);
        tokio::spawn(async move {
            if let Err(e) = proxy_conn(client, pool).await {
                eprintln!("proxy error: {e}");
            }
        });
    }
}
```

> [!TIP]
> For production L7 HTTP/1.1 + HTTP/2 reverse proxies, build on **`hyper`** (low-level HTTP engine) or Cloudflare's **`pingora`** rather than parsing raw HTTP headers by hand—subtle HTTP request-smuggling bugs (`Content-Length` vs `Transfer-Encoding`) live in hand-rolled parsers.

## Hands-On Lab: Verify Round-Robin Distribution Across Two Backends

Test the atomic round-robin index distribution with a zero-dependency Rust harness:

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

fn main() {
    let backends = ["10.0.0.1:8080", "10.0.0.2:8080", "10.0.0.3:8080"];
    let cursor = AtomicUsize::new(0);
    let mut counts = [0usize; 3];

    for _ in 0..900 {
        let idx = cursor.fetch_add(1, Ordering::Relaxed) % backends.len();
        counts[idx] += 1;
    }
    for (addr, count) in backends.iter().zip(counts) {
        println!("{addr} -> {count} requests");
    }
    assert_eq!(counts, [300, 300, 300]);
}
```
