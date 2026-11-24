---
title: Portable and Hardware Vector SIMD Scenarios (simd and simd/archsimd)
---

# Portable and Hardware Vector SIMD Scenarios (`simd` and `simd/archsimd`)

Enabled via `GOEXPERIMENT=simd` in Go 1.26–1.27, two complementary standard library packages expose hardware vector registers (AMD64 AVX2/AVX-512, ARM64 NEON, and 128-bit Wasm SIMD) directly through SSA without Plan 9 assembly or Cgo:

- **`simd` (Go 1.27):** Portable, vector-width-agnostic types (`simd.Float32s`, `simd.Uint8s`, `simd.Int32s`) that step by the CPU's native lane width and fall back cleanly.
- **`simd/archsimd` (Go 1.26–1.27):** Architecture-specific fixed-width vectors (`Float64x4`, `Int8x16`, `Float32x8`).

> [!NOTE]
> `simd` and `simd/archsimd` remain experimental behind `GOEXPERIMENT=simd` in Go 1.27.x. Benchmark against scalar code using `for b.Loop()` and `benchstat` before adopting in production hot paths.

## Scenario 1: Embedding Dot Product (`simd.Float32s`)

```go
// Build with: GOEXPERIMENT=simd go test ./...
package simdops

import "simd"

func DotProductSIMD(a, b []float32) float32 {
	lanes := simd.Float32s{}.Len()
	var acc simd.Float32s
	i := 0
	for ; i+lanes <= len(a); i += lanes {
		acc = acc.Add(simd.LoadFloat32s(a[i:]).Mul(simd.LoadFloat32s(b[i:])))
	}
	sum := acc.ReduceAdd()
	for ; i < len(a); i++ {
		sum += a[i] * b[i]
	}
	return sum
}
```

## Scenario 2: Branchless Telemetry Clamping and Newline Scanning

```go
// Build with: GOEXPERIMENT=simd
package simdops

import "simd"

func ClampSensorBatch(readings []float32, low, high float32) {
	lanes := simd.Float32s{}.Len()
	vLow, vHigh := simd.BroadcastFloat32s(low), simd.BroadcastFloat32s(high)
	i := 0
	for ; i+lanes <= len(readings); i += lanes {
		simd.LoadFloat32s(readings[i:]).Max(vLow).Min(vHigh).Store(readings[i:])
	}
	for ; i < len(readings); i++ {
		readings[i] = min(high, max(low, readings[i]))
	}
}

func CountNewlinesSIMD(buf []byte) int {
	lanes := simd.Uint8s{}.Len()
	needle := simd.BroadcastUint8s('\n')
	count, i := 0, 0
	for ; i+lanes <= len(buf); i += lanes {
		count += simd.LoadUint8s(buf[i:]).Equal(needle).CountTrue()
	}
	for ; i < len(buf); i++ {
		if buf[i] == '\n' {
			count++
		}
	}
	return count
}
```

## Related Chapters

- [186b SIMD with archsimd](../24-performance-engineering/186b-simd-with-archsimd.md)
- [230 ABI, Assembly, and nosplit](../25-go-deep-dives/230-abi-assembly-nosplit.md)
