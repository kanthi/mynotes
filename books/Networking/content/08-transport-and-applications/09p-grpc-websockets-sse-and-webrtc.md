---
title: 'gRPC, WebSockets, SSE, and WebRTC Media Paths'
---

# gRPC, WebSockets, SSE, and WebRTC Media Paths

Classic HTTP/1.1 request–response fits short-lived document fetches cleanly: a client opens a TCP connection, sends a request, receives a response, and any Layer 4 load balancer can spray new connections across backends. Modern distributed systems and real-time applications break that assumption. Microservices multiplex thousands of **gRPC** RPCs onto a single persistent HTTP/2 connection; dashboards push live updates over **WebSockets** or **Server-Sent Events (SSE)**; and voice, video, and low-latency data bypass TCP altogether over **WebRTC** UDP media paths. Each protocol shifts where state lives on the wire—and introduces its own load-balancing, proxy-timeout, and NAT-traversal traps.

## Learning goals

By the end of this chapter you can:

- Explain how gRPC frames Protocol Buffers inside HTTP/2 `DATA` frames and why HTTP/2 multiplexing breaks Layer 4 load balancing
- Choose between L7 proxies (Envoy/HAProxy), client-side load balancing (xDS/headless DNS), and `MAX_CONNECTION_AGE` for gRPC services
- Contrast WebSockets and Server-Sent Events (SSE) on the wire, including upgrade headers, proxy buffering, and idle-timeout traps
- Trace a WebRTC session end to end: signaling (SDP offer/answer), ICE candidate gathering and connectivity checks, and DTLS-SRTP media encryption
- Identify when WebRTC media traverses NAT directly via STUN versus falling back to a TURN relay

## Protocol comparison at a glance

| Protocol | Transport | Direction | Framing & Multiplexing | L4 Load Balancer Behavior | Primary Use Case |
|----------|-----------|-----------|------------------------|---------------------------|------------------|
| **gRPC** | TCP + TLS (HTTP/2) | Unary, server/client/bidi streaming | 5-byte length-prefixed protobuf in HTTP/2 streams | Pins all RPCs on one TCP socket to a single backend | Internal RPCs, service meshes, streaming APIs |
| **WebSockets** | TCP + TLS (HTTP/1.1 Upgrade or RFC 8441 H2) | Full-duplex bidirectional | Binary/text frames with client→server masking | Pins long-lived TCP session; requires graceful drain | Chat, collaborative editing, trading feeds |
| **SSE** | TCP + TLS (HTTP/1.1 or HTTP/2) | Server → Client only | UTF-8 `text/event-stream` (`data:`, `id:`, `event:`) | Per-connection on H1.1; multiplexed on H2 | Live logs, LLM token streaming, notifications |
| **WebRTC** | UDP (ICE / DTLS / SRTP / SCTP) | Full-duplex P2P or via SFU | SRTP packets (media) + SCTP over DTLS (data channels) | Not HTTP; uses ICE + STUN/TURN media relays | Video/audio calls, ultra-low-latency media |

## gRPC over HTTP/2

gRPC is not a separate transport protocol; it is a strict profile of **HTTP/2** (RFC 9113) carrying binary **Protocol Buffers** payloads. Every RPC maps to one HTTP/2 stream inside a shared TCP+TLS connection:

- **Request headers (`HEADERS` frame):** `:method: POST`, `:path: /pkg.Service/Method`, `content-type: application/grpc`, `te: trailers`, and optional `grpc-timeout`.
- **Payload (`DATA` frames):** Each gRPC message begins with a 5-byte prefix: 1 byte compression flag (`0x00` uncompressed, `0x01` compressed) + 4-byte big-endian message length, followed by the protobuf bytes. Multiple messages can flow in either direction on the same stream for streaming RPCs.
- **Status (`HEADERS` frame with `END_STREAM`):** gRPC conveys application errors in **HTTP/2 trailers** (`grpc-status: 0`, `grpc-message`), *not* the initial HTTP `:status: 200`. A reverse proxy that strips trailers breaks gRPC error handling completely.

### Flow control and head-of-line blocking

HTTP/2 enforces credit-based **flow control** at two levels: per-stream (`WINDOW_UPDATE` on a stream ID) and per-connection (`WINDOW_UPDATE` on stream 0). A slow consumer on one RPC stops sending `WINDOW_UPDATE` for its stream, exerting backpressure without stalling other streams—*unless* the connection-level window exhausts or a single TCP packet drops. Because all streams share one TCP socket, one lost TCP segment stalls delivery of every multiplexed stream until TCP retransmits it (transport-level head-of-line blocking, which QUIC solves; see [QUIC v2 and Version Negotiation](06-quic-v2-and-version-negotiation.md)).

### The L4 load-balancing trap

A Layer 4 load balancer (NLB, IPVS, Maglev, or Kubernetes `kube-proxy` iptables/IPVS) makes its backend decision **once per TCP 4-tuple** during the `SYN` handshake.

```text
  gRPC Client                     L4 Load Balancer                Backends
  (1 persistent HTTP/2 conn)                                   ┌─▶ Pod A (100% of RPCs!)
  ───────────────────────────────▶ [hash(src,dst)] ────────────┤
   Stream 1, 2, 3 ... 10,000                                   ├─  Pod B (0% idle)
                                                               └─  Pod C (0% idle)
```

When a gRPC client opens one long-lived HTTP/2 connection, every RPC—even 10,000 requests per second—lands on **Pod A**. Worse, when autoscaling adds Pods D, E, and F during a traffic spike, existing clients never open new TCP connections, so the new pods sit completely idle while Pod A melts.

### Fixing gRPC load balancing

There are three production patterns to distribute gRPC traffic evenly:

1. **Layer 7 reverse proxy (Envoy, HAProxy, NGINX, Linkerd/Istio):** The L7 proxy terminates HTTP/2 from the client, inspects each individual HTTP/2 stream, and dispatches each RPC to a different backend over pooled upstream HTTP/2 connections.
2. **Client-side load balancing (xDS or headless DNS):** The gRPC client resolves all backend pod IPs (via a Kubernetes headless Service or an xDS control plane), opens a connection pool to *every* backend, and runs `round_robin` or `weighted_round_robin` per RPC with zero proxy hop.
3. **Server-enforced connection rotation (`MAX_CONNECTION_AGE`):** Even behind an L4 balancer, configuring gRPC servers with `GRPC_ARG_MAX_CONNECTION_AGE_MS` (for example 5 minutes + random jitter) causes the server to send an HTTP/2 `GOAWAY` frame periodically. In-flight RPCs finish cleanly, and the client opens a fresh TCP connection that re-hashes across newly scaled backends.

> [!TIP]
> Always combine `MAX_CONNECTION_AGE` with `MAX_CONNECTION_AGE_GRACE` and jitter (`+/- 10%`). Without jitter, all clients reconnect in lockstep and create a thundering herd of TLS handshakes.

## WebSockets vs Server-Sent Events (SSE)

Both WebSockets and SSE keep an HTTP-originated connection open for server-initiated pushes, but their wire mechanics and operational failure modes differ sharply.

### WebSockets (RFC 6455)

A browser opens a WebSocket by sending an HTTP/1.1 `Upgrade` request:

```http
GET /ws/ticks HTTP/1.1
Host: stream.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

If the server agrees, it replies `101 Switching Protocols`. From that exact byte onward, the TCP socket **stops speaking HTTP** and switches to raw binary WebSocket frames (opcode, payload length, and a mandatory 4-byte XOR mask on all client→server frames to prevent proxy cache poisoning).

### Server-Sent Events (SSE)

SSE needs no protocol upgrade at all. It is a plain HTTP `GET` whose response has `Content-Type: text/event-stream` and never closes:

```http
HTTP/2 200
content-type: text/event-stream
cache-control: no-cache

id: 4091
event: token
data: {"chunk":"hello"}

id: 4092
event: token
data: {"chunk":" world"}
```

Browsers implement SSE natively via `EventSource`, which **automatically reconnects** on network drops and sends `Last-Event-ID: 4092` so the server can replay missed events seamlessly. Over HTTP/2, multiple SSE streams multiplex over a single TCP connection; over HTTP/1.1, browsers cap concurrent connections per origin (typically 6), which can starve other requests if you open many SSE tabs.

### Proxy upgrade, buffering, and timeout traps

Long-lived WebSockets and SSE streams frequently die mysteriously behind reverse proxies (NGINX, Envoy, ALB, Cloudflare). Four misconfigurations cause 90% of outages:

- **Stripped hop-by-hop headers (WebSockets):** `Connection` and `Upgrade` are hop-by-hop headers in HTTP/1.1. A reverse proxy drops them by default unless explicitly configured (`proxy_http_version 1.1; proxy_set_header Upgrade $http_upgrade; proxy_set_header Connection "upgrade";`).
- **Response buffering (SSE):** Proxies buffer small response chunks to optimize compression and TCP segments. For SSE, buffering holds back events for seconds or minutes until a 4–8 KB buffer fills. Disable buffering explicitly (`X-Accel-Buffering: no` for NGINX, and disable gzip/brotli on `text/event-stream`).
- **Idle read timeouts (both):** Default proxy idle timeouts (often 60 seconds) silently kill quiet WebSocket or SSE sockets. Send application-level heartbeats every 15–30 seconds: WebSocket `PING`/`PONG` control frames (`0x9`/`0xA`), or SSE comment lines (`: keepalive\n\n`).
- **Abrupt deploys:** Restarting a backend drops every pinned WebSocket/SSE client at once, triggering a reconnect storm. Drain connections gradually with randomized reconnect backoff.

## WebRTC end-to-end media path

TCP's reliable, in-order delivery is poison for live audio and video: waiting 250 ms to retransmit a lost video frame stalls playback far worse than dropping the frame and moving on. **WebRTC** therefore runs real-time media over **UDP** (falling back to TCP/TLS only when UDP is blocked), combining four layers into one media path:

```text
  Peer A (behind NAT 1)                                     Peer B (behind NAT 2)
    │                                                                 │
    │◀══════════ 1. Signaling (HTTPS / WebSocket / SIP) ═════════════▶│
    │              exchange SDP Offer/Answer + ICE candidates         │
    │                                                                 │
    ├── 2. ICE Gather: host, srflx (via STUN), relay (via TURN) ──────┤
    │                                                                 │
    │◀── 3. ICE Connectivity Checks (STUN Binding req/resp pairs) ───▶│
    │                                                                 │
    │◀── 4. DTLS Handshake (over winning UDP 5-tuple, verify SDP fp) ▶│
    │                                                                 │
    │◀══ 5. SRTP Media (audio/video) + SCTP/DTLS (DataChannel) ══════▶│
```

### 1. Signaling and SDP offer/answer

WebRTC deliberately does **not** standardize signaling—applications exchange **SDP (Session Description Protocol)** blobs over whatever channel they already have (typically HTTPS or WebSockets). The SDP carries:
- Media descriptions (`m=audio`, `m=video`) and negotiated codecs (Opus, VP9, AV1, H.264)
- **ICE credentials** (`a=ice-ufrag`, `a=ice-pwd`) used to authenticate connectivity checks
- **DTLS certificate fingerprint** (`a=fingerprint:sha-256 ...`) that binds the media encryption handshake to the signaling channel, preventing an on-path attacker from MITMing the UDP stream

### 2. ICE candidate gathering and NAT traversal

Before media can flow, each peer runs **ICE (Interactive Connectivity Establishment, RFC 8445)** to discover every reachable transport address (**candidates**), ranked by priority:

1. **Host candidates:** Local interface IPs (`192.168.1.20:54102`), ideal when both peers share a LAN.
2. **Server-reflexive (`srflx`) candidates:** Public IP:port learned by querying a **STUN** server, enabling direct UDP hole punching across endpoint-independent NATs (see [NAT Traversal: STUN, Hole Punching and When You Need a Relay](../07-a-overlays-tunnels/08-nat-traversal-stun-hole-punching.md)).
3. **Relayed (`relay`) candidates:** Public IP:port allocated on a **TURN** server—the guaranteed fallback when symmetric NATs or strict corporate firewalls block direct UDP hole punching (see [TURN Relays: Credentials, Permissions and Peer Policy](../07-a-overlays-tunnels/08b-turn-relays-and-peer-policy.md)).

Peers exchange candidates incrementally (**Trickle ICE**) so connectivity checks start immediately instead of waiting for every STUN/TURN timeout to finish. Both sides test candidate pairs in priority order using authenticated STUN Binding requests until a bidirectional pair succeeds and is nominated.

### 3. DTLS-SRTP and RTCP feedback

Once ICE nominates a UDP 5-tuple, WebRTC **never** sends unencrypted media:

- **DTLS (Datagram TLS)** runs directly over the nominated UDP socket. Both peers verify that the remote certificate matches the `a=fingerprint` received in SDP.
- **SRTP / SRTCP (Secure Real-time Transport Protocol, RFC 3711)** derives symmetric session keys from the DTLS handshake via key export (`use_srtp` extension). Audio and video frames travel as encrypted SRTP packets; **RTCP** feedback packets (NACK, PLI, Transport-Wide Congestion Control `TWCC`) let senders adapt bitrate dynamically to packet loss and jitter.
- **DataChannels** run **SCTP** encapsulated inside DTLS, offering configurable reliability (ordered/reliable like TCP, or partial-reliability/unordered for game state).

> [!NOTE]
> Because STUN, DTLS, SRTP, and SRTCP all share a single UDP port (**bundle + rtcp-mux**), WebRTC demultiplexes incoming packets by inspecting the first byte of the UDP payload (RFC 7983): `0–3` = STUN, `20–63` = DTLS, `128–191` = RTP/RTCP.

### P2P vs SFU architectures

Direct peer-to-peer WebRTC works well for 1:1 calls, but full-mesh P2P collapses at 5+ participants because every client must encode and upload `N - 1` video streams. Multi-party conferencing therefore places a **SFU (Selective Forwarding Unit)** in the middle: each participant maintains one WebRTC session to the SFU, uploads a single simulcast/SVC stream, and the SFU decrypts/re-encrypts (or forwards with end-to-end insertable streams) the appropriate video layer to each receiver.

## Related chapters

- [QUIC v2 and Version Negotiation](06-quic-v2-and-version-negotiation.md)
- [HTTP/3 Stream Cancellation](09-http3-stream-cancellation.md)
- [Caddy Slowloris and Idle Timeouts](10-caddy-slowloris-idle-timeouts.md)
- [Load Balancer Hashing and Maglev](16-load-balancer-hashing-maglev.md)
- [NAT Traversal: STUN, Hole Punching and When You Need a Relay](../07-a-overlays-tunnels/08-nat-traversal-stun-hole-punching.md)
- [TURN Relays: Credentials, Permissions and Peer Policy](../07-a-overlays-tunnels/08b-turn-relays-and-peer-policy.md)
