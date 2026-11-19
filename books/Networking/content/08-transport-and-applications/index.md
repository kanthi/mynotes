---
title: Intro
---

# Intro

**Transport, TLS, and Applications** — Explore deep transport internals (TCP state machine, congestion control, SACK/RACK, UDP batching, QUIC), cryptographic protocols (TLS 1.3, ECH, PKI, DNSSEC, DANE/MTA-STS), and application delivery (HTTP/2/3, gRPC, WebSockets, WebRTC, load balancers, proxies).

## How to use this part

In the sidebar this folder is organized into three focused parts: **Transport Internals (TCP, UDP, QUIC)**, **TLS, PKI, and Cryptographic Protocols**, and **HTTP, Proxies, and Traffic Delivery**.

## Chapters in this part

| Chapter | File |
|---------|------|
| [QUIC Version 2 and Compatible Version Negotiation](06-quic-v2-and-version-negotiation.md) | `06-quic-v2-and-version-negotiation.md` |
| [QUIC Loss Recovery: Packet Threshold, Time Threshold and PTO (RFC 9002)](06b-quic-loss-recovery.md) | `06b-quic-loss-recovery.md` |
| [Happy Eyeballs: Racing IPv6 and IPv4 (RFC 8305 and the v3 Draft)](07-happy-eyeballs-v2-v3.md) | `07-happy-eyeballs-v2-v3.md` |
| [QUIC Load Balancing with nftables: Routing on Connection IDs](08-quic-load-balancing-with-nftables.md) | `08-quic-load-balancing-with-nftables.md` |
| [HTTP/3 Stream Cancellation: STOP_SENDING, RESET_STREAM, and 499](09-http3-stream-cancellation.md) | `09-http3-stream-cancellation.md` |
| [TCP States, Reproduced: A Two-Namespace Lab](09a-tcp-states-lab.md) | `09a-tcp-states-lab.md` |
| [Header Compression in HTTP/2 and HTTP/3: HPACK and QPACK from the Wire Up](09b-hpack-qpack-header-compression.md) | `09b-hpack-qpack-header-compression.md` |
| [Small Writes in TCP: Nagle''s Algorithm, Delayed ACKs, and the 40 ms Stall](09b-tcp-nagle-and-delayed-ack.md) | `09b-tcp-nagle-and-delayed-ack.md` |
| [HTTP Request Smuggling: CL.TE, TE.CL and Obfuscated Transfer-Encoding](09c-http-request-smuggling.md) | `09c-http-request-smuggling.md` |
| [TCP Fast Open](09c-tcp-fast-open.md) | `09c-tcp-fast-open.md` |
| [TIME-WAIT on Linux: What the Timers Really Do](09d-tcp-time-wait.md) | `09d-tcp-time-wait.md` |
| [Zero-Copy Sends with MSG_ZEROCOPY: Completions, Deferred Copies, and When](09e-msg-zerocopy-sends.md) | `09e-msg-zerocopy-sends.md` |
| [SO_REUSEPORT: Many Listeners on One Port](09f-so-reuseport-listener-groups.md) | `09f-so-reuseport-listener-groups.md` |
| [Multipath TCP: Subflows, Path Managers and a Rootless Lab](09g-multipath-tcp-subflows-lab.md) | `09g-multipath-tcp-subflows-lab.md` |
| [TCP Congestion Control: CUBIC, Reno and BBR](09h-tcp-congestion-cubic-bbr.md) | `09h-tcp-congestion-cubic-bbr.md` |
| [Where the Microseconds Go: Timestamping One UDP Datagram Through the Linux](09i-udp-latency-budget-timestamping.md) | `09i-udp-latency-budget-timestamping.md` |
| [Dead Peers and Silent Paths: TCP Keepalive, TCP_USER_TIMEOUT and tcp_retries2](09j-tcp-dead-peer-detection.md) | `09j-tcp-dead-peer-detection.md` |
| [TCP Loss Recovery on Linux: RTO, Fast Retransmit, SACK, RACK and Tail Loss](09k-tcp-loss-recovery-sack-rack-tlp.md) | `09k-tcp-loss-recovery-sack-rack-tlp.md` |
| [TCP Windows, Buffers and the Bandwidth-Delay Product: Autotuning, Buffer Caps](09l-tcp-windows-buffers-bdp.md) | `09l-tcp-windows-buffers-bdp.md` |
| [Batching UDP on Linux: sendmmsg, recvmmsg, UDP GSO and GRO](09m-udp-batching-mmsg-gso-gro.md) | `09m-udp-batching-mmsg-gso-gro.md` |
| [Choosing a Source Port: connect(), bind() and the Ephemeral Range](09n-source-port-selection-ephemeral-range.md) | `09n-source-port-selection-ephemeral-range.md` |
| [Blind Resets and Challenge ACKs: RFC 5961 on Linux](09o-tcp-rfc5961-challenge-acks.md) | `09o-tcp-rfc5961-challenge-acks.md` |
| [gRPC, WebSockets, SSE, and WebRTC Media Paths](09p-grpc-websockets-sse-and-webrtc.md) | `09p-grpc-websockets-sse-and-webrtc.md` |
| [Caddy 2.11.6: Slowloris idle timeouts and url_pattern matchers](10-caddy-slowloris-idle-timeouts.md) | `10-caddy-slowloris-idle-timeouts.md` |
| [Inside the TLS 1.3 Key Schedule: HKDF, Three Secrets, and Every Key in a Handshake](10-tls13-key-schedule.md) | `10-tls13-key-schedule.md` |
| [The 2026 Root KSK Rollover: Trust Anchors, RFC 5011, and What Breaks When](11-root-ksk-rollover-rfc5011.md) | `11-root-ksk-rollover-rfc5011.md` |
| [TLS 1.3 0-RTT: Replaying Early Data, and What Actually Stops It](12-tls13-0rtt-replay.md) | `12-tls13-0rtt-replay.md` |
| [TLS Session Resumption in Practice: Tickets, Ticket Keys and Fleets](12b-tls-session-resumption-and-ticket-keys.md) | `12b-tls-session-resumption-and-ticket-keys.md` |
| [Encrypting IP Addresses in Logs with IPCrypt](13-ipcrypt-log-privacy.md) | `13-ipcrypt-log-privacy.md` |
| [Encrypted Client Hello: Hiding the Server Name](14-encrypted-client-hello.md) | `14-encrypted-client-hello.md` |
| [DNSSEC Failure Modes: SERVFAIL, Extended DNS Errors, Serve-Stale and NTAs](15-dnssec-failure-modes.md) | `15-dnssec-failure-modes.md` |
| [DANE and TLSA: Pinning TLS Keys in DNSSEC](15b-dane-tlsa-smtp.md) | `15b-dane-tlsa-smtp.md` |
| [MTA-STS and TLSRPT: Enforcing TLS for Mail Without DNSSEC](15c-mta-sts-and-tlsrpt.md) | `15c-mta-sts-and-tlsrpt.md` |
| [Load Balancer Hashing: Modulo, Rings, Rendezvous and Maglev](16-load-balancer-hashing-maglev.md) | `16-load-balancer-hashing-maglev.md` |
| [Merkle Tree Certificates: Keeping Post-Quantum TLS Handshakes Small](20-merkle-tree-certificates.md) | `20-merkle-tree-certificates.md` |
| [Who May Issue Your Certificate: CAA, ACME Challenges and Revocation After](20b-caa-acme-and-revocation.md) | `20b-caa-acme-and-revocation.md` |
| [X.509 Path Building Traps: Intermediates, nameConstraints and pathlen](20c-x509-path-building-name-constraints.md) | `20c-x509-path-building-name-constraints.md` |
| [Auditing DNS Against NIST SP 800-81r3](23-dns-nist-sp-800-81r3-audit.md) | `23-dns-nist-sp-800-81r3-audit.md` |
| [The PROXY Protocol: Keeping the Client Address Through an L4 Proxy](25-proxy-protocol.md) | `25-proxy-protocol.md` |
| [Forward Proxies: HTTP CONNECT, SOCKS5 and the Proxy Environment Variables](26-forward-proxies-connect-socks.md) | `26-forward-proxies-connect-socks.md` |
