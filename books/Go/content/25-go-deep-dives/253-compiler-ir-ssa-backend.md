---
title: Compiler IR, Escape Analysis, SSA Backend, and Bounds-Check Elimination
---

# Compiler IR, Escape Analysis, SSA Backend, and Bounds-Check Elimination

`cmd/compile` is a purpose-built, pure-Go compiler engineered for fast builds and predictable machine code without an LLVM backend. Understanding its 5-stage pipeline, escape analysis flow graph, SSA passes, and Bounds-Check Elimination (BCE) lets you write idiomatic Go that compiles to bare-metal loops.

## The Five Stages of `cmd/compile`

```text
Go Source Code (.go)
         |
         v
[1. Lexing & Parsing] (cmd/compile/internal/syntax)     -> Concrete Syntax Tree
         |
         v
[2. Type Checking] (cmd/compile/internal/types2)        -> Typed AST & Generic Inference
         |
         v
[3. IR & Escape Analysis] (internal/noder, escape)      -> Stack vs. Heap Placement
         |
         v
[4. SSA Optimization] (cmd/compile/internal/ssa)        -> 40+ Passes (DCE, CSE, BCE, Lower)
         |
         v
[5. Code Generation] (cmd/compile/internal/genssa)      -> AMD64 / ARM64 / RISC-V / Wasm
```

## Escape Analysis Flow Graph (`internal/escape`)

Escape analysis builds a weighted directed graph of variables (`location`) and assignments (`hole`):
- Direct assignment (`a = b`): weight `0`.
- Address-of (`a = &b`): weight `+1`.
- Dereference (`a = *b`): weight `-1`.

Five invariants force heap escape: returning a pointer to a local, capture by a long-lived closure, assignment to an interface (unless devirtualized), dynamic sizes exceeding stack thresholds (note Go 1.25–1.26 `variablemake` keeps small variable `make([]T, n)` on the stack), and sending pointers over channels.

## SSA Form, Phi Nodes, and Bounds-Check Elimination (BCE)

```text
        +-----------------+
        |    x0 = 10      |
        +--------+--------+
                 |
         if condition == true
        /                  \
    [True]                [False]
      v                      v
+-----------+          +-----------+
|  x1 = 20  |          |  x2 = 30  |
+-----+-----+          +-----+-----+
      \                      /
       +----------+---------+
                  |
                  v
        +-------------------+
        | x3 = phi(x1, x2)  |  <-- SSA Phi Node
        +-------------------+
```

The SSA `prove` pass tracks integer inequality ranges to eliminate redundant `runtime.panicIndex` checks:

```go
func processArray(s []int) int {
	_ = s[3] // Single upfront bounds check proves len(s) > 3.
	return s[0] + s[1] + s[2] + s[3] // All 4 subsequent checks eliminated!
}
```

## Inspecting Compiler Decisions and Go 1.26–1.27 Upgrades

```bash
GOSSAFUNC=processArray go build main.go          # Generates interactive ssa.html
go build -gcflags="-m -m" main.go                # Escape analysis & inlining reasons
go build -gcflags="-d=ssa/check_bce" main.go     # Bounds-check insertion report
```

- **Go 1.27 size-specialized malloc lowering:** SSA lowers small (`< 80 B`) allocations directly to specialized routines (up to 30% faster).
- **Go 1.27 generic method lowering & closure code sharing:** Lowers `(*Rand).N[Int]` with full inference and deduplicates inlined closure bodies.
- **PGO (Profile-Guided Optimization):** Raises the 80-point inlining budget on hot call edges (`go build -pgo=default.pgo`) for 2%–14% production CPU gains.
- **Ecosystem SSA tools:** Pair [gohawk](https://gohawk.dev/) (interprocedural SSA `resourcelifetime`, `lockorder`, `goroutineownership`) with `noalloc` (`//noalloc:root`) and [lensm](https://github.com/loov/lensm) for disassembly inspection.

## Related Chapters

- [209 Compiler, SSA, and PGO](209-compiler-ssa-pgo.md)
- [252 Compiler Frontend](252-compiler-frontend.md)
- [186d Inside PGO Compiler Internals](../24-performance-engineering/186d-inside-pgo-compiler-internals.md)
