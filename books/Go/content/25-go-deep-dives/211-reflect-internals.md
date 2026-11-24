---
title: Reflect, internal/abi.Type Metadata, and Zero-Alloc Type Introspection
---

# Reflect, `internal/abi.Type` Metadata, and Zero-Alloc Type Introspection

Go is statically compiled with no virtual machine, yet `encoding/json/v2`, `database/sql`, and `fmt` inspect arbitrary struct fields, tags, and methods at runtime via `reflect`. Between Go 1.21 and Go 1.27, the runtime unified type descriptors in `src/internal/abi/type.go`, added zero-allocation `reflect.TypeAssert[T]` (Go 1.25), and introduced rangeable `Type.Fields()` / `Value.Fields()` iterators (Go 1.26).

## The Unified Type Descriptor: `internal/abi.Type` (48 Bytes in `.rodata`)

`reflect.Type`, `eface._type`, `itab._type`, and the Green Tea GC all share the exact same immutable 48-byte `abi.Type` header emitted into `.rodata`:

```text
                    internal/abi.Type (48 Bytes in .rodata)
+-------------------------------------------------------------------------------+
| Size_:       uintptr (8B)  | Exact byte size of concrete type (unsafe.Sizeof) |
| PtrBytes:    uintptr (8B)  | Highest byte offset containing a GC pointer      |
| Hash:        uint32  (4B)  | Type hash for O(1) itab & TypeAssert checks      |
| TFlag:       TFlag   (1B)  | Uncommon, ExtraStar, Named, RegularMemory        |
| Align_:      uint8   (1B)  | Memory alignment (1, 2, 4, 8, 16)                |
| FieldAlign_: uint8   (1B)  | Alignment when embedded as a struct field        |
| Kind_:       Kind    (1B)  | Reflect kind + KindDirectIface bit (0x20)        |
| Equal:       func(p, q unsafe.Pointer) bool (8B) | Compiler-generated equal   |
| GCData:      *byte   (8B)  | Bitmask telling Green Tea GC which words are ptrs|
| Str:         NameOff (4B)  | Relative 32-bit offset into .rodata name table   |
| PtrToThis:   TypeOff (4B)  | Relative 32-bit offset to *T's abi.Type          |
+-------------------------------------------------------------------------------+
```

- **`PtrBytes` cuts GC scan work:** Putting pointer fields at the top of a struct minimizes `PtrBytes` so the Green Tea GC stops scanning before trailing scalar fields. If `PtrBytes == 0`, the type routes to `noscan` spans.
- **`KindDirectIface` (`1 << 5`):** Indicates types (`*T`, `chan T`, `map[K]V`, `func()`, single-pointer structs) stored directly in `eface.data` without extra heap indirection.
- **Relative 32-bit offsets (`NameOff`, `TypeOff`):** Eliminate dynamic linker relocations in `-buildmode=pie` binaries.

## Anatomy of `reflect.Value` (24 Bytes) and Bitpacked `flag`

```go
type Value struct {
	typ_ *abi.Type      // Word 0: immutable type descriptor in .rodata
	ptr  unsafe.Pointer // Word 1: pointer to value (or direct value if flagIndir == 0)
	flag                // Word 2: bitmask for Kind, Addressability, & Read-Only state
}
```

| Bit Range in `flag` | Constant | Role |
|---|---|---|
| Bits `0..4` | `flagKindMask` (`31`) | `reflect.Kind` for `O(1)` switch dispatch |
| Bit `5` | `flagStickyRO` | Obtained via unexported struct field (`Set()` panics) |
| Bit `6` | `flagEmbedRO` | Obtained via unexported embedded struct pointer |
| Bit `7` | `flagIndir` | `1` if `ptr` points to value memory; `0` if `ptr` *is* the value |
| Bit `8` | `flagAddr` | Addressable (`CanAddr() == true`); required for `Set()` |
| Bit `9` | `flagMethod` | Bound method value (`v.Method(i)`) |

## Zero-Allocation Extraction: `reflect.TypeAssert[T]` (Go 1.25+)

Legacy `v.Interface().(time.Time)` boxes `v` into an `any` (`eface`) on the heap before asserting. In Go 1.25–1.27, `reflect.TypeAssert[T](v)` compares `v.typ_ == abi.TypeFor[T]()` (one pointer compare) and copies directly into the caller's stack return slot (`0 B/op`, ~87% faster):

```go
// Go 1.25–1.27: 0 B/op, 0 allocs/op.
if ts, ok := reflect.TypeAssert[time.Time](v); ok {
	_ = ts
}
```

> [!TIP]
> Pass `reflect.ValueOf(&cfg).Elem()` instead of `reflect.ValueOf(cfg)` so non-pointer structs avoid boxing copies and gain `flagAddr` addressability. Use `reflect.TypeFor[T]()` (Go 1.22+) instead of `reflect.TypeOf((*T)(nil)).Elem()`.

## Cached Struct-Offset Plan with Go 1.26 `Type.Fields()`

Production codecs compute `field.Offset` once per `reflect.Type` (cached in Go 1.24+'s `HashTrieMap` `sync.Map`) and write directly via `unsafe.Add`:

```go
package fastcodec

import (
	"reflect"
	"sync"
	"unsafe"
)

type fieldPlan struct {
	name   string
	offset uintptr
	kind   reflect.Kind
}

var planCache sync.Map // map[reflect.Type][]fieldPlan

func CompilePlan(t reflect.Type) []fieldPlan {
	if cached, ok := planCache.Load(t); ok {
		return cached.([]fieldPlan)
	}
	var plans []fieldPlan
	// Go 1.26+: range-over-func iterator avoids copying 104-byte StructField manually.
	for field := range t.Fields() {
		if !field.IsExported() {
			continue
		}
		tag := field.Tag.Get("env")
		if tag == "" || tag == "-" {
			continue
		}
		plans = append(plans, fieldPlan{
			name:   tag,
			offset: field.Offset,
			kind:   field.Type.Kind(),
		})
	}
	actual, _ := planCache.LoadOrStore(t, plans)
	return actual.([]fieldPlan)
}

func PopulateFromEnv[T any](target *T, lookup func(string) (string, bool)) {
	t := reflect.TypeFor[T]()
	plans := CompilePlan(t)
	basePtr := unsafe.Pointer(target)
	for _, fp := range plans {
		if fp.kind != reflect.String {
			continue
		}
		if val, ok := lookup(fp.name); ok {
			*(*string)(unsafe.Add(basePtr, fp.offset)) = val
		}
	}
}
```

Verify zero allocations with `noalloc` and [benchstat](https://pkg.go.dev/golang.org/x/perf/cmd/benchstat), inspect `LEAQ type:T(SB)` lowering in [lensm](https://github.com/loov/lensm), and audit `MethodByName` dead-code impact with `go build -ldflags="-dumpdep"`.

## Related Chapters

- [204 Interface Representation](204-interface-representation.md)
- [232 Build Cache, Action DAG, and Linker Internals](232-build-cache-reproducibility.md)
- [01 Reflection in Practice](../26-long-term-maintenance/01-reflection-in-practice.md)
