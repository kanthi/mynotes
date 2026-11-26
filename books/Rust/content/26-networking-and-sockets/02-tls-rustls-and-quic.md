---
title: Memory-Safe TLS with rustls and QUIC with quinn
---

# Memory-Safe TLS with `rustls` and QUIC with `quinn`

For decades, network services linked against C OpenSSL—inheriting buffer-overflow CVEs, fragile cross-compilation, and legacy TLS 1.0/1.1 negotiation quirks. Rust's networking ecosystem replaces C TLS stacks with **`rustls`** (a pure-Rust TLS 1.2/1.3 implementation backed by `aws-lc-rs` or `ring`) and **`quinn`** (a pure-Rust QUIC transport powering HTTP/3).

## Why `rustls` Is the Production Default

- **Strict modern defaults:** TLS 1.2 and TLS 1.3 only, forward-secret AEAD cipher suites (AES-GCM, ChaCha20-Poly1305), zero SSLv3/RC4/MD5 legacy attack surface.
- **Zero C toolchain friction:** Cross-compiling static `x86_64-unknown-linux-musl` or `aarch64` binaries works out of the box without installing OpenSSL system headers.
- **Pluggable certificate verification:** Supports `webpki-roots` (embedded Mozilla root bundle) or `rustls-native-certs` (OS trust store for corporate CAs).

## Configuring a `tokio-rustls` TLS 1.3 Server

```rust
use std::fs::File;
use std::io::BufReader;
use std::sync::Arc;
use tokio_rustls::rustls::{self, pki_types::CertificateDer};
use tokio_rustls::TlsAcceptor;

pub fn build_tls_acceptor(cert_path: &str, key_path: &str) -> std::io::Result<TlsAcceptor> {
    let mut cert_reader = BufReader::new(File::open(cert_path)?);
    let certs: Vec<CertificateDer<'static>> =
        rustls_pemfile::certs(&mut cert_reader).collect::<Result<_, _>>()?;

    let mut key_reader = BufReader::new(File::open(key_path)?);
    let key = rustls_pemfile::private_key(&mut key_reader)?
        .ok_or_else(|| std::io::Error::new(std::io::ErrorKind::InvalidData, "missing key"))?;

    let mut cfg = rustls::ServerConfig::builder()
        .with_no_client_auth()
        .with_single_cert(certs, key)
        .map_err(|e| std::io::Error::new(std::io::ErrorKind::InvalidData, e))?;

    // Advertise HTTP/2 and HTTP/1.1 via ALPN
    cfg.alpn_protocols = vec![b"h2".to_vec(), b"http/1.1".to_vec()];
    Ok(TlsAcceptor::from(Arc::new(cfg)))
}
```

## QUIC and HTTP/3 with `quinn`

HTTP/2 over TCP still suffers from **transport-layer head-of-line (HoL) blocking**: if a single TCP packet drops on a lossy mobile link, the OS stalls delivery of *every* multiplexed HTTP/2 stream until that packet retransmits. **QUIC** moves congestion control, TLS 1.3 encryption, and independent multiplexed streams onto UDP:

| Feature | TCP + TLS 1.3 (HTTP/2) | QUIC (`quinn`, HTTP/3) |
| :--- | :--- | :--- |
| **Handshake latency** | 1 RTT (TCP) + 1 RTT (TLS) = 2 RTT | 1 RTT combined (0-RTT resumption supported) |
| **Head-of-line blocking** | Packet loss stalls all streams on connection | Each stream reassembles independently |
| **Connection migration** | Switching Wi-Fi → 5G breaks TCP 4-tuple | Connection ID survives IP address changes |
| **CPU cost** | Mature kernel TCP offload (GSO/GRO) | Higher userspace UDP cost unless using GSO batching |

> [!WARNING]
> Only allow QUIC **0-RTT early data** for strictly idempotent read requests (`GET` / `HEAD`). 0-RTT packets can be replayed by an on-path attacker before the server completes the full handshake.

## Hands-On Lab: Generate Local Self-Signed Certs and Verify ALPN

Generate a local development certificate with `openssl` and verify TLS 1.3 handshake parameters against your `tokio-rustls` listener:

```bash
mkdir -p /tmp/rust-tls-lab && cd /tmp/rust-tls-lab
openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 \
  -keyout server.key -out server.crt -days 7 -nodes \
  -subj "/CN=localhost" -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"

# Inspect the generated ECDSA P-256 certificate
openssl x509 -in server.crt -noout -subject -dates -ext subjectAltName
```
