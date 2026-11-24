---
title: Machine Representation & Addressability
---

# Machine Representation & Addressability

Understanding how Go's basic types map onto machine memory—and which expressions have a memory address—explains subtle compiler errors, struct padding, floating-point edge cases, and pointer behavior.

## Integer Types: Bit-Width, Two's Complement, and Architecture

Fixed-width integers (`int8`, `int16`, `int32`, `int64` and their `uint` counterparts) always occupy the exact number of bits in their name, regardless of CPU architecture.

Signed integers use **two's complement** representation:
- The most significant bit (MSB) is the sign bit (`0` for non-negative, `1` for negative).
- Overflow **wraps around** silently in Go (modulo $2^N$); there is no runtime overflow panic.

The architecture-dependent types `int` and `uint` match the platform's native word size:
- **32-bit targets** (`GOARCH=386`, `arm`): `int` and `uint` are 32 bits (4 bytes).
- **64-bit targets** (`GOARCH=amd64`, `arm64`): `int` and `uint` are 64 bits (8 bytes).

`uintptr` is an unsigned integer large enough to hold any pointer bit pattern on the target architecture. Unlike `*T`, `uintptr` is just a number—the garbage collector does **not** treat it as a live pointer.

## Floating-Point Types: IEEE-754 Semantics

Go's `float32` and `float64` implement **IEEE-754** binary floating-point arithmetic:

| Type | Total Bits | Sign | Exponent | Mantissa (Fraction) |
|------|-----------|------|----------|---------------------|
| `float32` | 32 | 1 bit | 8 bits (bias 127) | 23 bits |
| `float64` | 64 | 1 bit | 11 bits (bias 1023) | 52 bits |

The value of a normal number is:

```text
(-1)^sign * (1 + mantissa) * 2^(exponent - bias)
```

### Special IEEE-754 Values and Traps

- **Signed Zero (`+0.0` and `-0.0`):** `0.0 == -0.0` evaluates to `true`, even though their sign bits differ (`math.Signbit(-0.0)` is `true`).
- **Infinities (`+Inf`, `-Inf`):** Produced by overflow or dividing a non-zero float by zero (`1.0 / 0.0` at runtime; `math.Inf(1)`).
- **`NaN` (Not a Number):** Produced by `0.0 / 0.0` or `math.Sqrt(-1)`. By IEEE-754 rules, **`NaN != NaN` is always `true`**—use `math.IsNaN(x)` instead of `x == x`.
- **Precision loss:** Base-10 fractions like `0.1` have no exact binary representation, so `0.1 + 0.2 != 0.3`. Use integer minor units (cents) or decimal libraries for money.

```go
package main

import (
	"fmt"
	"math"
)

func main() {
	var f float32 = -0.0
	fmt.Printf("bits=0x%08x signbit=%t eqZero=%t\n",
		math.Float32bits(f), math.Signbit(float64(f)), f == 0.0)

	nan := math.NaN()
	fmt.Printf("NaN == NaN: %t, IsNaN: %t\n", nan == nan, math.IsNaN(nan))
}
```

## Boolean Type: 1-Byte Memory Representation

Even though a `bool` only holds one bit of information (`true` or `false`), **`unsafe.Sizeof(bool(true))` is `1` byte** (8 bits).

Modern CPUs are **byte-addressable**, not bit-addressable—every addressable variable must occupy at least one full byte so a pointer `*bool` can point to it directly.

> [!TIP]
> Because `bool` takes 1 byte and has 1-byte alignment, placing several `bool` fields next to each other in a struct packs them together instead of wasting alignment padding between 8-byte fields.

## Addressable and Unaddressable Values

In Go, you can only apply the address-of operator (`&x`) to **addressable** values.

### What Is Addressable?

1. **Variables** (`x := 10; p := &x`)
2. **Pointer indirections** (`*p`)
3. **Slice indexing** (`s[i]`, because slices have a contiguous backing array)
4. **Array indexing of an addressable array** (`arr[i]`)
5. **Struct field selection of an addressable struct** (`st.Field`)
6. **Composite literal exception:** `&MyStruct{}` is syntactic sugar that allocates a struct and takes its address.

### What Is Unaddressable?

1. **Constants** (`&myConst` fails to compile)
2. **Map indexing** (`&m[key]` fails to compile—Go's map implementation can rehash and move buckets in memory, which would invalidate pointers to map values)
3. **Function return values** (`&fn()` fails)
4. **String indexing** (`&s[0]` fails—strings are immutable)
5. **Values inside an interface** (`&ifaceVal` takes the address of the interface variable, not the boxed concrete value)
6. **Type conversions** (`&int64(x)` fails)

### Why Addressability Matters

- **Pointer-receiver methods:** If `T` has a method `func (t *T) Mutate()`, you can call `v.Mutate()` on an **addressable** value `v` of type `T` (Go automatically inserts `(&v).Mutate()`). Calling `Mutate()` on an **unaddressable** value like `m["k"].Mutate()` or `getT().Mutate()` fails to compile.
- **Reflection:** `reflect.Value.CanAddr()` and `reflect.Value.CanSet()` both require the underlying value to be addressable (for example, passed via `reflect.ValueOf(&x).Elem()`).

## Related Chapters

- [09 Predeclared Types](09-predeclared-types.md)
- [23 Pointers](../03-memory/23-pointers.md)
- [25c Escape Analysis & Alignment](../03-memory/25c-escape-analysis-alignment.md)
- [205 Slice & Map Internals](../25-go-deep-dives/205-slice-map-internals.md)
- [211 Reflect Internals](../25-go-deep-dives/211-reflect-internals.md)
