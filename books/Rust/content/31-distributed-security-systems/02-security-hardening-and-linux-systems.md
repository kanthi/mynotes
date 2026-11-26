---
title: Security Hardening, Cryptography, and Linux Systems Programming
---

# Security Hardening, Cryptography, and Linux Systems Programming

Memory safety eliminates roughly 70% of historical systems-software CVEs (buffer overflows, use-after-free, double-free), but secure Rust engineering goes further: **constant-time cryptography**, **explicit secret zeroization**, **supply-chain auditing**, and modern Linux kernel interfaces (**`io_uring`** and **eBPF with `aya`**).

## Modern Rust Security & Crypto Stack

| Need | Recommended Crate | Why |
| :--- | :--- | :--- |
| **AEAD encryption** | `chacha20poly1305` / `aes-gcm` | Authenticated encryption prevents ciphertext tampering |
| **Password hashing** | `argon2` (Argon2id) | Memory-hard, resistant to GPU/ASIC brute force |
| **Secret memory hygiene** | `zeroize` + `secrecy` | Overwrites key bytes on `Drop` using volatile writes LLVM won't optimize away |
| **Constant-time equality** | `subtle` (`ConstantTimeEq`) | Prevents timing side channels when comparing MACs/tokens |
| **Supply-chain checks** | `cargo-audit` + `cargo-deny` | Flags RustSec advisories, disallowed licenses, and duplicate crates in CI |

## Why `zeroize` Matters Against Compiler Optimizations

If you manually overwrite a `[u8; 32]` key buffer with zeros at the end of a function, LLVM sees that the buffer is never read again and **deletes the zeroing loop** as dead code. The `zeroize` crate uses `core::ptr::write_volatile` and a compiler memory barrier so secret material is guaranteed to be wiped from RAM when the struct drops.

## Linux Systems Programming: `io_uring` and eBPF (`aya`)

- **`io_uring` (`tokio-uring`, `monoio`, `glommio`):** Replaces per-operation `read`/`write` syscalls with lock-free shared-memory **Submission Queues (SQ)** and **Completion Queues (CQ)** between userspace and the Linux kernel—unlocking true asynchronous file I/O and sub-microsecond batching.
- **eBPF in Pure Rust (`aya`):** Lets you write both the kernel-space eBPF probe (`#![no_std]`) and the userspace loader/map reader in a single Cargo workspace—sharing exact `#[repr(C)]` event structs between kernel and userspace with zero C boilerplate.

> [!IMPORTANT]
> Run `cargo audit` and `cargo clippy -- -D warnings` on every pull request, and isolate any `unsafe` FFI wrappers inside small, audited internal modules with explicit `// SAFETY:` comments documenting every upheld invariant.

## Hands-On Lab: Constant-Time Comparison Check

Verify why constant-time byte comparison avoids early-exit timing leaks:

```rust
fn constant_time_eq(a: &[u8], b: &[u8]) -> bool {
    if a.len() != b.len() {
        return false;
    }
    let mut diff = 0u8;
    for (&x, &y) in a.iter().zip(b.iter()) {
        diff |= x ^ y;
    }
    diff == 0
}

fn main() {
    assert!(constant_time_eq(b"digest-abc123", b"digest-abc123"));
    assert!(!constant_time_eq(b"digest-abc123", b"digest-abc124"));
    println!("Constant-time digest comparison passed.");
}
```
