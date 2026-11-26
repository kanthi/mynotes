---
title: WebSockets, Server-Sent Events, and Graceful Shutdown
---

# WebSockets, Server-Sent Events, and Graceful Shutdown

Real-time dashboards, live log tailing, and streaming LLM responses require long-lived connections. Because Tokio tasks occupy only a few hundred bytes of memory each, a single Rust backend instance can sustain tens of thousands of concurrent WebSocket or Server-Sent Events (SSE) streams—provided you handle **slow consumers** and **graceful shutdown** cleanly.

## SSE vs. WebSockets

| Protocol | Direction | Transport | Reconnect Built-In? | Best Fit |
| :--- | :--- | :--- | :--- | :--- |
| **SSE (`text/event-stream`)** | Server → Client | Plain HTTP/1.1 or HTTP/2 | Yes (`EventSource` + `Last-Event-ID`) | LLM token streaming, live metrics, notifications |
| **WebSockets (`ws://` / `wss://`)** | Full-duplex bidirectional | HTTP Upgrade | No (client must implement backoff) | Chat, collaborative editing, multiplayer state |

## Fan-Out Broadcast with Bounded Lag Handling

Use `tokio::sync::broadcast::channel(capacity)` to fan out events from a producer to many connected clients. If one client stalls on a slow network, `broadcast` returns `RecvError::Lagged(skipped)` instead of blocking fast subscribers or growing memory unboundedly:

```rust
use axum::{
    extract::{
        ws::{Message, WebSocket, WebSocketUpgrade},
        State,
    },
    response::IntoResponse,
};
use tokio::sync::broadcast;

async fn ws_handler(
    ws: WebSocketUpgrade,
    State(tx): State<broadcast::Sender<String>>,
) -> impl IntoResponse {
    ws.on_upgrade(move |socket| handle_socket(socket, tx.subscribe()))
}

async fn handle_socket(mut socket: WebSocket, mut rx: broadcast::Receiver<String>) {
    loop {
        tokio::select! {
            msg = rx.recv() => match msg {
                Ok(text) => {
                    if socket.send(Message::Text(text.into())).await.is_err() {
                        break; // Client disconnected
                    }
                }
                Err(broadcast::error::RecvError::Lagged(n)) => {
                    eprintln!("slow subscriber dropped {n} messages");
                }
                Err(broadcast::error::RecvError::Closed) => break,
            },
            incoming = socket.recv() => match incoming {
                Some(Ok(Message::Close(_))) | None => break,
                _ => {}
            }
        }
    }
}
```

## Zero-Downtime Graceful Shutdown on `SIGTERM`

When Kubernetes sends `SIGTERM` during a rolling deploy, abruptly exiting kills in-flight requests and corrupts active streams. Wire `axum::serve(...).with_graceful_shutdown(...)` to OS signals so the listener stops accepting new connections while allowing existing requests to drain:

```rust
pub async fn shutdown_signal() {
    let ctrl_c = async {
        tokio::signal::ctrl_c().await.expect("failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        tokio::signal::unix::signal(tokio::signal::unix::SignalKind::terminate())
            .expect("failed to install SIGTERM handler")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }
}
```

> [!TIP]
> Pair `with_graceful_shutdown` with a `tokio_util::sync::CancellationToken` shared with long-lived WebSocket/SSE loops so streaming handlers close cleanly within Kubernetes's `terminationGracePeriodSeconds` window.

## Hands-On Lab: Verify `broadcast` Lag Behavior Without Slowing Fast Readers

```rust
use std::sync::mpsc;

// Pure-std mental model check: bounded ring behavior vs lagging consumer
fn main() {
    let capacity = 2;
    let mut ring = std::collections::VecDeque::with_capacity(capacity);
    for event in ["e1", "e2", "e3", "e4"] {
        if ring.len() == capacity {
            let dropped = ring.pop_front().unwrap();
            println!("evicted lagged event: {dropped}");
        }
        ring.push_back(event);
    }
    println!("surviving events in ring: {ring:?}");
    let _ = mpsc::channel::<()>();
}
```
