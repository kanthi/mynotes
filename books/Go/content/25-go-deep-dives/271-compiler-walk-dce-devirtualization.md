---
title: Compiler Walk Pass, Dead Code Elimination, and Devirtualization
---

# Compiler Walk Pass, Dead Code Elimination, and Devirtualization

Between type-checking/escape analysis and SSA code generation, `cmd/compile` runs several critical IR transformations—most notably the **`walk` pass**, **Dead Code Elimination (DCE)**, and **devirtualization**. Understanding these stages explains what "magic" syntax like `range`, `make`, and `select` actually compiles into, and why some interface calls inline while others pay dynamic dispatch costs.

## Where `walk` Fits in the Pipeline

```text
  AST / Typecheck (types2)
          |
          v
  Unified IR + Inlining + Escape Analysis + Devirtualization
          |
          v
  Walk (cmd/compile/internal/walk)   <-- Desugars high-level Go ops into runtime calls
          |
          v
  SSA (cmd/compile/internal/ssa)     <-- 40+ passes (prove, deadcode, cse, lower)
          |
          v
  Machine Code (genssa) + Linker
```

## Stage: The `walk` Pass (`cmd/compile/internal/walk`)

Before SSA can optimize a function, every high-level Go language construct must be lowered into primitive operations and explicit calls to `runtime` functions. That is `walk`'s job.

| Go Source Construct | Lowered by `walk` Into |
|---|---|
| `make([]T, n)` (heap-escaped) | `runtime.makeslice` |
| `append(s, x)` (when `cap` exceeded) | `runtime.growslice` |
| `s1 + s2 + s3` | `runtime.concatstrings` |
| `m[k]` (map lookup) | `runtime.mapaccess1` / `mapaccess2` |
| `select { ... }` | `runtime.selectgo` (see [249 selectgo](249-selectgo-compiler-runtime.md)) |
| `var x any = v` | `runtime.convT2E` / `convT64` / `convTstring` |
| `for i, v := range s` | Index loop (`i := 0; i < len(s); i++`) |

`walk` also inserts **order-of-evaluation** temporaries, `nil` checks, and write-barrier hooks where needed, ensuring the resulting IR is a clean, explicit sequence of basic operations ready for SSA.

## Dead Code Elimination (DCE)

Go eliminates unreachable code at **three** distinct levels:

### 1. Constant-Branch Pruning (Frontend / IR)

When an `if` condition is a compile-time constant—such as `const debug = false` or `runtime.GOOS == "windows"` on Linux—the compiler drops the unreachable branch entirely before SSA. That means imports, allocations, or functions referenced *only* inside the dead branch are never compiled into the binary.

```go
const enableVerbose = false

func handle() {
	if enableVerbose {
		// Completely eliminated at compile time; zero runtime cost.
		expensiveDebugDump()
	}
}
```

### 2. SSA `deadcode` Pass (`cmd/compile/internal/ssa`)

During SSA optimization, passes like `prove` (value-range analysis) and `opt` simplify branches. Any basic block that becomes unreachable—or any computed value whose result is never used and has no side effects—is stripped by the `deadcode` pass.

### 3. Linker Dead-Code Elimination (`cmd/link`)

Even after compilation, `cmd/link` builds a reachability graph starting from `main.main` and `init` functions and discards unreferenced functions and symbols.

> [!NOTE]
> Using `reflect.Value.MethodByName` or `//go:linkname` can force the linker to keep more methods in the binary because static reachability cannot prove which methods are called dynamically.

## Devirtualization

Calling a method through an interface (`r.Read(buf)`) normally requires an indirect call through the interface's `itab` method table, which blocks inlining and often forces the receiver to escape to the heap.

**Devirtualization** rewrites an interface call into a direct call to the concrete type's method.

### Static Devirtualization

When the compiler can statically see the concrete type assigned to an interface in the same function (or after inlining), it replaces the `itab` dispatch with a direct call:

```go
type Greeter interface {
	Greet() string
}

type English struct{}
func (English) Greet() string { return "hello" }

func run() string {
	var g Greeter = English{} // Concrete type is statically known
	return g.Greet()          // Devirtualized -> English.Greet() -> Inlined!
}
```

### PGO Speculative Devirtualization

With Profile-Guided Optimization (`go build -pgo=default.pgo`), even when the concrete type is *not* statically known (e.g., `io.Reader` passed into a helper), if the profile shows 95%+ of calls hit `*os.File` or `*bytes.Reader`, the compiler emits a fast type-check guard:

```text
if r.(*bytes.Reader) {
    // Direct, inlined call to (*bytes.Reader).Read
} else {
    // Fallback to normal interface dispatch
}
```

### Inspecting Devirtualization and Inlining

```bash
go build -gcflags="-m -m" ./...
```

Look for compiler diagnostics such as:
- `devirtualizing g.Greet() to English`
- `inlining call to English.Greet`

## Related Chapters

- [209 Compiler, SSA, and PGO](209-compiler-ssa-pgo.md)
- [252 Compiler Frontend](252-compiler-frontend.md)
- [253 Compiler IR, Escape Analysis, SSA Backend, and Bounds-Check Elimination](253-compiler-ir-ssa-backend.md)
- [186d Inside PGO Compiler Internals](../24-performance-engineering/186d-inside-pgo-compiler-internals.md)
