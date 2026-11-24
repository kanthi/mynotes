---
title: Build Cache, Action DAG, MVS, and Linker Internals (.gopclntab)
---

# Build Cache, Action DAG, MVS, and Linker Internals (`.gopclntab`)

Turning a multi-module Go workspace into a reproducible static executable is orchestrated by `cmd/go`'s concurrent build action graph (`src/cmd/go/internal/work`), Minimal Version Selection (`src/cmd/go/internal/mvs`), and the Go linker (`src/cmd/link/internal/ld`).

## `cmd/go` Build Action DAG and `__.PKGDEF` Export-Data Hashing

```text
                cmd/go Concurrent Build Action DAG (cmd/go/internal/work)
+-------------------------------------------------------------------------------+
|                    [ Action: link (cmd/link -> final binary) ]                |
|                                        ^                                      |
|                                        | (Consumes .a object archives)        |
|                    [ Action: build main (cmd/compile) ]                       |
|                         /                              \                      |
|      (Consumes ONLY Export Data __.PKGDEF, NOT full _go_.o implementation)    |
|                       v                                  v                    |
|     [ Action: build pkg/auth ]                  [ Action: build pkg/db ]      |
+-------------------------------------------------------------------------------+
```

When `cmd/compile` compiles `pkg/db`, it emits a `.a` archive containing:

1. **`__.PKGDEF` (Export Data):** Exported type signatures, constants, and inlinable function bodies.
2. **`_go_.o` (Object Code):** Compiled machine instructions.

To decide whether `main` needs recompilation, `cmd/go` hashes `main`'s sources, flags, and **only** `pkg/db`'s `__.PKGDEF` hash—so editing a non-inlined function body inside `pkg/db` skips recompiling `main` and jumps straight to `cmd/link`.

## Go 1.24–1.27 Toolchain Upgrades

- **`GOCACHEPROG` (Go 1.24 GA):** Point `GOCACHEPROG=/path/to/helper` at a JSON-over-stdio child process (`get`, `put`, `close` keyed by `ActionID`) to back `GOCACHE` with S3, GCS, or Redis in CI.
- **`go.mod` `tool` directives (Go 1.24–1.27):** Register CLI tools with `go get -tool github.com/kojah/gohawk@latest` and execute cached binaries via `go tool gohawk ./...`—replacing legacy `tools.go` hacks.
- **`go build -json` / `go test -json`:** Emit structured build and diagnostic events for IDEs and CI.
- **Minimal Version Selection (MVS):** Resolves the maximum of the minimum versions declared in the `go.mod` graph in a single deterministic `O(V + E)` walk—no backtracking SAT solver.

## Linker Internals (`cmd/link`): Dead-Code Elimination and `.gopclntab`

1. **`loader.Loader`:** Assigns compact 32-bit `loader.Sym` indices across all `.a` archives without loading full symbol bodies into RAM.
2. **Reachability dead-code elimination (`deadcode`):** Walks from `_rt0_*` -> `runtime.rt0_go` -> package `init()` -> `main.main`.
   > [!WARNING]
   > **The `reflect.Value.MethodByName` trap:** Calling `MethodByName(dynamicName)` with a non-constant string forces `deadcode` to mark **every exported method on every reachable type** as live, bloating binaries by 20%–40%. Audit with `go build -ldflags="-dumpdep"`.
3. **Relocation-free `.gopclntab` (Go 1.26–1.27):** Stores `funcnametab`, `filetab`, delta-compressed `pctab`, and `O(log N)` binary-searchable `functab` using relative 32-bit offsets in `.rodata` with zero dynamic relocations—powering stack unwinding, `copystack`, Green Tea GC stack maps, and `deferBits`.

### Inspecting `.gopclntab` in Go 1.27

```go
package main

import (
	"debug/elf"
	"debug/gosym"
	"fmt"
	"os"
)

func main() {
	exePath, err := os.Executable()
	if err != nil {
		panic(err)
	}
	f, err := elf.Open(exePath)
	if err != nil {
		return // Non-ELF platform (e.g., Mach-O on macOS uses debug/macho).
	}
	defer f.Close()

	pclndat, _ := f.Section(".gopclntab").Data()
	textSect := f.Section(".text")
	lineTable := gosym.NewLineTable(pclndat, textSect.Addr)
	symTable, _ := gosym.NewTable(nil, lineTable)

	if fn := symTable.LookupFunc("main.main"); fn != nil {
		file, line, _ := symTable.PCToLine(fn.Entry)
		fmt.Printf("main.main PC: 0x%x -> %s:%d (.gopclntab: %d KB)\n",
			fn.Entry, file, line, len(pclndat)/1024)
	}
}
```

## Reproducible Build Recipe

```bash
CGO_ENABLED=0 go build -trimpath -buildvcs=true -ldflags="-s -w -buildid=" -o dist/app ./cmd/app
```

Passing `-ldflags="-s -w"` strips the OS symbol table and DWARF5 debug sections (~28% smaller binary) while preserving `.gopclntab` and `.go.buildinfo` so panic stack traces and `pprof` keep full function names and line numbers.

## Related Chapters

- [248 Runtime Bootstrap](248-runtime-bootstrap.md)
- [265 Why Go Binaries Are Big](265-binary-size-linker.md)
- [326 CLI Packaging, Self-Updating Binaries, and ML-DSA](../09-cli-tools-in-go/326-cli-packaging-self-update-mldsa.md)
