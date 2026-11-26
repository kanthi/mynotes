---
title: Understanding no_std and Bare-Metal Rust
---

# Understanding `#![no_std]` and Bare-Metal Rust

Rust's standard library (`std`) assumes an operating system exists underneath your program—providing threads, filesystems, TCP sockets, environment variables, and a heap allocator. Microcontrollers, OS kernels, bootloaders, and eBPF programs have none of those. Adding `#![no_std]` at the top of a crate detaches your binary from `std` while keeping Rust's full ownership, trait, and zero-cost abstraction guarantees intact.

## The Three-Tier Standard Library Hierarchy

| Crate | Requires OS? | Requires Heap Allocator? | What It Provides |
| :--- | :--- | :--- | :--- |
| **`core`** | No | No | Primitives, `Option`, `Result`, `Iterator`, slices, atomics, `Pin`, `Future`, formatting traits |
| **`alloc`** | No | Yes (`#[global_allocator]`) | `Box<T>`, `Vec<T>`, `String`, `Arc<T>`, `BTreeMap` |
| **`std`** | Yes | Yes | Re-exports `core` + `alloc`, plus `fs`, `net`, `thread`, `io`, `env`, `time` |

## What a `#![no_std]` Binary Must Provide

When you opt out of `std` in an executable (not just a library), two runtime hooks normally supplied by `std` become your responsibility:

1. **A `#[panic_handler]` function:** Defines what the hardware does on an unrecoverable panic (for example, halting in an infinite loop, flashing an LED, or logging via RTT/defmt).
2. **A custom entry point (`#![no_main]`):** Because there is no OS C runtime (`crt0`) calling `main()`, your reset vector or crate macro (like `#[cortex_m_rt::entry]`) becomes the first instruction executed.

```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;

// Fixed-capacity ring buffer requiring zero heap allocation
pub struct RingBuffer<const N: usize> {
    buf: [u8; N],
    head: usize,
    tail: usize,
}

impl<const N: usize> RingBuffer<N> {
    pub const fn new() -> Self {
        Self { buf: [0; N], head: 0, tail: 0 }
    }

    pub fn push(&mut self, byte: u8) -> Result<(), u8> {
        let next = (self.head + 1) % N;
        if next == self.tail {
            return Err(byte); // Buffer full
        }
        self.buf[self.head] = byte;
        self.head = next;
        Ok(())
    }
}

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {
        core::hint::spin_loop();
    }
}
```

> [!TIP]
> When authoring reusable crates (parsers, crypto, protocol encoders), make them `#![no_std]` by default and gate `std`-only conveniences behind a Cargo `std` feature (`#![cfg_attr(not(feature = "std"), no_std)]`). That unlocks embedded and WebAssembly users at almost zero extra cost.

## Hands-On Lab: Verify a `#![no_std]` Library Compiles Cleanly

Check that a `core`-only library has zero accidental `std` imports using `rustc`:

```bash
cat > /tmp/nostd_ring.rs <<'RS'
#![no_std]

pub fn checksum(bytes: &[u8]) -> u32 {
    bytes.iter().fold(0u32, |acc, &b| acc.wrapping_add(b as u32))
}
RS

rustc --crate-type=lib --edition=2021 /tmp/nostd_ring.rs -o /tmp/libnostd_ring.rlib
ls -lh /tmp/libnostd_ring.rlib
```
