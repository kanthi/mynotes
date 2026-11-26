---
title: Redis Caching, Stampede Prevention, and Embedded Storage
---

# Redis Caching, Stampede Prevention, and Embedded Storage

Database read replicas scale only so far before hot keys saturate CPU. Production Rust services combine **distributed caching** (Redis/Valkey via `redis` or `fred`), **in-process caches** (`moka` lock-free TinyLFU), and **embedded ACID storage** (`redb` or `sqlite`) when single-binary zero-ops persistence is the right tool.

## Cache-Aside Pattern and Cache Stampede Prevention

In **cache-aside**, the application checks the cache first; on a miss, it queries PostgreSQL, populates the cache with a jittered TTL, and returns the value. Under high concurrency, when a popular key expires simultaneously for 500 incoming requests, all 500 miss the cache and hammer the database at once—a **cache stampede** (thundering herd).

Two simple defenses eliminate stampedes:

1. **TTL jitter:** Add random jitter (`base_ttl + rand(0..base_ttl/10)`) so keys populated in a batch do not expire at the exact same second.
2. **Request coalescing (singleflight):** Track in-flight key fetches in a shared map of `tokio::sync::watch` or `broadcast` channels so only **one** task queries PostgreSQL while the other 499 await its result.

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use std::time::{Duration, Instant};

struct CacheEntry<V> {
    value: V,
    expires_at: Instant,
}

pub struct TtlCache<K, V> {
    inner: Arc<Mutex<HashMap<K, CacheEntry<V>>>>,
}

impl<K: Eq + std::hash::Hash, V: Clone> TtlCache<K, V> {
    pub fn new() -> Self {
        Self { inner: Arc::new(Mutex::new(HashMap::new())) }
    }

    pub fn get(&self, key: &K) -> Option<V> {
        let mut map = self.inner.lock().unwrap();
        if let Some(entry) = map.get(key) {
            if Instant::now() < entry.expires_at {
                return Some(entry.value.clone());
            }
            map.remove(key);
        }
        None
    }

    pub fn insert(&self, key: K, value: V, ttl: Duration) {
        let mut map = self.inner.lock().unwrap();
        map.insert(key, CacheEntry { value, expires_at: Instant::now() + ttl });
    }
}
```

## Embedded Storage in Pure Rust (`redb` vs. SQLite)

| Engine | Language | Concurrency | Best Fit |
| :--- | :--- | :--- | :--- |
| **`redb`** | 100% pure Rust | MVCC copy-on-write B-tree (multiple readers, 1 writer) | Embedded KV stores, local caches, CLI state, edge appliances |
| **`rusqlite` / `sqlx-sqlite`** | C (bundled) + Rust wrapper | WAL mode concurrent reads + single writer | Complex relational queries, full-text search (FTS5), single-node apps |
| **RocksDB (`rust-rocksdb`)** | C++ + Rust FFI | High write-throughput LSM-tree | Write-heavy time-series, blockchain state, storage engines |

> [!TIP]
> For in-process bounded caching with automatic TinyLFU eviction, TTL expiration, and built-in `get_with()` stampede coalescing, use the **`moka`** crate rather than hand-rolling an unbounded `HashMap`.

## Hands-On Lab: Verify TTL Expiration in `TtlCache`

```rust
use std::thread;
use std::time::Duration;

fn main() {
    let cache = TtlCache::new();
    cache.insert("session:42", "active", Duration::from_millis(50));
    assert_eq!(cache.get(&"session:42"), Some("active"));

    thread::sleep(Duration::from_millis(70));
    assert_eq!(cache.get(&"session:42"), None);
    println!("TTL expiration verified cleanly.");
}
```
