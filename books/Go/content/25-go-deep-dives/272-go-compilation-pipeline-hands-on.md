---
title: '272. Inspecting the Go Compilation Pipeline: Source to Assembly'
---

# Inspecting the Go Compilation Pipeline: Source to Assembly

Every `go build` transforms Go source code through seven well-defined compiler stages before emitting machine code. Knowing which command-line flag exposes each stage lets you verify inlining, escape analysis, devirtualization, SSA passes, and final assembly directly instead of guessing.

## The Seven Compiler Stages

```text
Go Source (.go)
   |
   v  Stage 1: Syntax Tree (Lexing & Parsing -> cmd/compile/internal/syntax)
   v  Stage 2: Type Checking (types2: generics, constraints, method sets)
   v  Stage 3: IR Generation (noder -> cmd/compile/internal/ir)
   v  Stage 4: Optimizations (inlining, devirtualization, dead code elimination, escape analysis)
   v  Stage 5: Walk (lowering high-level ops: maps, channels, defer, range -> runtime calls)
   v  Stage 6: Static Single Assignment (SSA passes, bounds-check elimination, register allocation)
   v  Stage 7: Architecture-Specific Assembly & Object Emission (amd64, arm64, ...)
```

## Stage-by-Stage Inspection Commands

| Stage | What Happens | Command to Inspect |
|-------|--------------|--------------------|
| **1. Syntax Tree** | Tokenizes source and builds an AST | `go run` with `go/parser` + `ast.Print`, or compiler `-W` |
| **2. Type Checking** | Resolves identifiers, checks types & generic constraints | `go vet ./...` or `go build -gcflags="-d=checkptr"` |
| **3. IR Generation** | Converts typed syntax into compiler IR nodes | `go build -gcflags="-W=1" ./...` |
| **4. Optimizations** | Inlining, devirtualization, DCE, escape analysis | `go build -gcflags="-m=2" ./...` |
| **5. Walk** | Rewrites `make(map)`, `chan`, `defer`, `append` into runtime calls | `go build -gcflags="-W=2" ./...` |
| **6. SSA Form** | Generates `ssa.html` showing every SSA optimization pass | `GOSSAFUNC=MyFunc go build ./...` |
| **7. Arch Assembly** | Lowers SSA to target CPU instructions (`amd64` / `arm64`) | `go build -gcflags="-S" ./...` or `go tool objdump -s MyFunc ./app` |

## Practical Walkthrough: Watching One Function Through Every Stage

Consider a small function that tests inlining, escape analysis, and bounds-check elimination:

```go
package main

func sumFirstThree(xs []int) int {
	if len(xs) < 3 {
		return 0
	}
	return xs[0] + xs[1] + xs[2]
}
```

Run the inspection flags side by side:

```bash
# Stage 4: see inlining cost, escape analysis, and bounds-check elimination
go build -gcflags="-m=2 -d=ssa/prove/debug=1" .

# Stage 6: generate interactive side-by-side SSA pass viewer (opens ssa.html)
GOSSAFUNC=sumFirstThree go build .

# Stage 7: inspect Plan 9 pseudo-assembly for amd64 or cross-compile for arm64
GOARCH=arm64 go build -gcflags="-S" . 2>&1 | sed -n '/^main.sumFirstThree/,/^$/p'
```

## Key Insights Across Stages

- **Stage 4 determines allocation performance:** Escape analysis runs *after* inlining and devirtualization. Inlining a small callee or devirtualizing an interface call often lets the compiler prove a pointer never escapes to the heap.
- **Stage 5 (`walk`) is where Go syntax meets `src/runtime`:** High-level statements like `m[k] = v`, `ch <- x`, and `defer f()` become concrete calls to `runtime.mapassign`, `runtime.chansend1`, and open-coded defer slots.
- **Stage 6 (`GOSSAFUNC`) is the ground truth for micro-optimizations:** Before rewriting readable Go code for speed, check `ssa.html`—the `prove` and `cse` passes frequently eliminate redundant bounds checks and branches automatically.

## Related Chapters

- [252. Compiler Frontend (Scan, Parse, Types)](252-compiler-frontend.md)
- [253. Compiler IR, SSA, and Backend](253-compiler-ir-ssa-backend.md)
- [271. Compiler Walk, Dead Code Elimination, and Devirtualization](271-compiler-walk-dce-devirtualization.md)
- [209. Compiler SSA and PGO](209-compiler-ssa-pgo.md)
- [230. ABI, Assembly, and `nosplit`](230-abi-assembly-nosplit.md)
