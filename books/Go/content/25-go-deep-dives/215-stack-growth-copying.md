---
title: Stack Growth, Contiguous Stacks, Pointer Adjustment, and runtime/secret
---

# Stack Growth, Contiguous Stacks, Pointer Adjustment, and `runtime/secret`

OS threads allocate 2–8 MB upfront, making 100,000 threads impractical. Go launches every goroutine with a **2 KB contiguous stack** that doubles on demand (up to 1 GB on 64-bit, configurable via `debug.SetMaxStack`) and shrinks lazily during GC.

## Contiguous Stacks vs. Legacy Segmented Stacks

Go 1.0–1.2 used linked segmented stacks, which suffered the **hot-split problem**: a tight loop crossing a segment boundary allocated and freed a chunk on every iteration (10x–50x slowdown). Go 1.3+ allocates a single contiguous buffer of `2 * oldSize`, copies the entire stack, and frees the old buffer.

## Function Prologue Check and `runtime.morestack`

Every normal function prologue checks `SP` against `g.stackguard0` (`16(R14)` on amd64, where `R14` holds `*g`):

```text
TEXT "".myFunction(SB), ABIInternal, $40-8
    CMPQ SP, 16(R14)                 // Compare SP with g.stackguard0
    JLS  morestack                   // Branch to runtime.morestack_noctxt if SP <= stackguard0
    SUBQ $40, SP                     // Normal fast path: bump SP
    ...
    ADDQ $40, SP
    RET
morestack:
    CALL runtime.morestack_noctxt(SB)
    JMP  0                           // Restart function on the expanded stack
```

## Inside `runtime.newstack` and `copystack` (`src/runtime/stack.go`)

```text
     Old Stack (2 KB)                         New Stack (4 KB)
+------------------------+               +------------------------+
| Frame 1: main()        |  -- Copied -> | Frame 1: main()        |
+------------------------+               +------------------------+
| Frame 2: process()     |  -- Copied -> | Frame 2: process()     |
+------------------------+               +------------------------+
| Frame 3: calculate()   |  -- Copied -> | Frame 3: calculate()   |
+------------------------+               +------------------------+
                                         | Unused Space (2 KB)    |
                                         +------------------------+
```

1. **Switch to `g0`:** Stack growth runs on the OS thread's fixed 8 KB `g0` stack via `mcall`.
2. **Allocate & copy:** Small stacks (`< 32 KB`) come from per-`P` `mcache` pools without global locks.
3. **Pointer adjustment (`adjustframe`):** Using compiler-emitted stack pointer bitmaps (`FUNCDATA_LocalsPointerMaps`), any stack pointer `p` inside `[old.lo, old.hi)` is shifted by `new.lo - old.lo`—including pointers held in channel `sudog.elem` and `_defer` frames!
4. **Lazy shrinking during GC:** If active stack usage drops below **25%** (`1/4`) of capacity during GC marking, `runtime.shrinkstack` halves the buffer (never below 2 KB).

## The `//go:nosplit` Directive

Low-level runtime functions (`morestack`, scheduler lock holders, signal trampolines) use `//go:nosplit` to omit the prologue check. The compiler enforces a hard **792-byte** `nosplit` chain limit at compile time (`nosplit stack overflow`).

## Go 1.24–1.27 Stack and Escape Upgrades

- **Stack-allocated variable-size slices (Go 1.25–1.26):** Non-escaping `make([]T, n)` with runtime `n` uses bounded stack scratch space (diagnosable with `go tool bisect -compile=variablemake`).
- **`new(expr)` stack allocation (Go 1.26+):** Non-escaping `new(computeValue())` stays on the stack at zero GC cost.
- **`runtime/secret` stack & register zeroing (Go 1.26–1.27):** `GOEXPERIMENT=runtimesecret` securely zeroes registers, stack frames, and copied stack segments after secret execution; in Go 1.27, child goroutines spawned inside `secret.Do` inherit secret mode automatically.
- **`#cgo noescape` (Go 1.24+) & 30% faster Cgo transitions (Go 1.26+):** Keeps Cgo pointer arguments on the Go stack and reduces `g0` switch overhead.

## Related Chapters

- [25b Stack and Heap Mechanics](../03-memory/25b-stack-heap-mechanics.md)
- [179 runtime/secret Zeroization](../22-security-hardening/179-runtime-secret-zeroization.md)
- [230 ABI, Assembly, and nosplit](230-abi-assembly-nosplit.md)
