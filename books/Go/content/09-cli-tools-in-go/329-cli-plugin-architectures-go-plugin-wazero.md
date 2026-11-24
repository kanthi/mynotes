---
title: Plugin Architectures for Go CLIs (go-plugin, wazero Wasm, and Native plugin Pitfalls)
---

# Plugin Architectures for Go CLIs (`go-plugin`, `wazero` Wasm, and Native `plugin` Pitfalls)

Every mature developer CLI or platform daemon eventually needs extensibility—custom cloud providers (`terraform-provider-*`), custom subcommands (`kubectl-*`, `gh-*`), storage backends, or request filters—without recompiling the host Go binary. Because Go compiles statically to native machine code, architects choose among four distinct plugin models.

## Why Go's Native `plugin` Package (`-buildmode=plugin`) Fails in Production

```text
Host Binary (Go 1.27.x + dep v1.2.0)              Plugin .so (Go 1.27.y or dep v1.2.1)
+---------------------------------------------+    +---------------------------------------------+
| runtime.firstmoduledata                     |    | plugin.moduledata                           |
| - Global Green Tea GC heap & goroutines     |    | - plugininit() checks exact dependency hash |
+----------------------+----------------------+    +----------------------+----------------------+
                       |                                                  |
                       +------------> plugin.Open("p.so") <---------------+
                                              |
                                              v
  FATAL ERROR: "plugin was built with a different version of package internal/abi"
               OR "plugin was built with a different version of package github.com/foo/bar"
```

1. **Exact compiler & dependency hash lock:** `plugin_lastmoduleinit` (`src/runtime/plugin.go`) compares SHA-256 build hashes of every shared package. Even a patch-level Go bump breaks `plugin.Open`.
2. **Requires `CGO_ENABLED=1` and dynamic `libc`:** Breaks pure-Go static builds and Alpine `musl` vs. Debian `glibc` portability.
3. **Zero unloading:** Loaded `.so` segments can never be unloaded—hot reloads permanently leak memory.
4. **No crash isolation:** A panic or Cgo `SIGSEGV` inside a plugin kills the entire host process.
5. **No Windows support:** `-buildmode=plugin` is unsupported on `GOOS=windows`.

## Model 1: PATH-Based Executable Plugins (`kubectl` / `git` / `gh`)

When the root dispatcher finds no built-in subcommand matching `mycli db-migrate`, it searches `$PATH` via `exec.LookPath("mycli-db-migrate")` (with hyphenated fallback `mycli-db`) and runs the executable via `exec.CommandContext`, inheriting `os.Stdin`, `os.Stdout`, `os.Stderr`, and `os.Environ()`. Plugins can be written in any language and distributed independently.

## Model 2: Subprocess RPC Plugins (`hashicorp/go-plugin`)

When your CLI needs typed bidirectional interfaces and complete crash/version isolation (Terraform, Vault, Packer), [hashicorp/go-plugin](https://github.com/hashicorp/go-plugin) runs plugins as long-lived child OS processes:

```text
Host CLI Process                                  Child Plugin Subprocess (mycli-plugin-aws)
+-------------------------------------------+     +-------------------------------------------+
| 1. Spawns child via os/exec.Cmd           |     | 2. Checks Handshake MagicCookie in Env    |
| 3. Reads stdout line 1:                   | <== |    Prints: "1|1|unix|/tmp/plug.sock|grpc" |
|    "CORE_PROTO|APP_PROTO|NET|ADDR|TYPE"   |     | 4. Opens gRPC server over local Unix      |
| 5. Connects gRPC Client + yamux mux       | <=> |    Socket (or yamux over stdin/stdout)    |
| 6. Host & Plugin can crash independently  |     | 7. Logs structured JSON over stderr ->    |
|    Host streams stderr -> host slog       | <== |    automatically mirrored in host logs    |
+-------------------------------------------+     +-------------------------------------------+
```

## Model 3: Embedded WebAssembly Sandboxes (`wazero` + `GOOS=wasip1`)

When you need **100,000+ plugin calls/sec** with strict memory and CPU sandboxing and zero Cgo (`CGO_ENABLED=0`), [wazero](https://github.com/tetratelabs/wazero) compiles `.wasm` modules directly to native AMD64/ARM64 machine code in pure Go:

```text
Plugin Author (Go 1.27, Rust, Zig, C)
  GOOS=wasip1 GOARCH=wasm go build -buildmode=c-shared -o filter.wasm ./plugin
                                      |
                                      v
Host Go CLI (Zero Cgo; Pure Go + wazero Runtime)
+-------------------------------------------------------------------------------+
| 1. Compile module once at startup (WithMemoryLimitPages(256) -> 16 MiB cap)   |
| 2. Instantiate per-worker or pooled module (isolated linear memory []byte)    |
| 3. Invoke exported Wasm function in ~50–150 nanoseconds (no syscalls)         |
| 4. WithCloseOnContextDone(true) aborts infinite loops instantaneously         |
+-------------------------------------------------------------------------------+
```

Go 1.24–1.27 supports reactor/library builds (`-buildmode=c-shared` on `wasip1`) with `//go:wasmexport` and compact incremental Wasm heaps.

### Complete Sandboxed `wazero` Plugin Host in Go 1.27

```go
package wasmhost

import (
	"context"
	"fmt"
	"time"

	"github.com/tetratelabs/wazero"
	"github.com/tetratelabs/wazero/imports/wasi_snapshot_preview1"
)

// PluginHost executes untrusted Wasm plugins inside a memory-bounded,
// timeout-enforced pure-Go sandbox (CGO_ENABLED=0).
type PluginHost struct {
	runtime  wazero.Runtime
	compiled wazero.CompiledModule
}

func LoadPlugin(ctx context.Context, wasmBytes []byte, maxMemoryMB uint32) (*PluginHost, error) {
	// 1 Wasm page = 64 KiB -> 16 pages = 1 MiB.
	maxPages := maxMemoryMB * 16

	cfg := wazero.NewRuntimeConfig().
		WithMemoryLimitPages(maxPages).
		WithCloseOnContextDone(true)

	rt := wazero.NewRuntimeWithConfig(ctx, cfg)
	if _, err := wasi_snapshot_preview1.Instantiate(ctx, rt); err != nil {
		_ = rt.Close(ctx)
		return nil, fmt.Errorf("instantiate wasi: %w", err)
	}

	compiled, err := rt.CompileModule(ctx, wasmBytes)
	if err != nil {
		_ = rt.Close(ctx)
		return nil, fmt.Errorf("compile wasm module: %w", err)
	}

	return &PluginHost{runtime: rt, compiled: compiled}, nil
}

func (h *PluginHost) Close(ctx context.Context) error {
	return h.runtime.Close(ctx)
}

func (h *PluginHost) InvokeFilter(parentCtx context.Context, input []byte, timeout time.Duration) ([]byte, error) {
	ctx, cancel := context.WithTimeout(parentCtx, timeout)
	defer cancel()

	mod, err := h.runtime.InstantiateModule(ctx, h.compiled, wazero.NewModuleConfig().WithName(""))
	if err != nil {
		return nil, fmt.Errorf("instantiate plugin: %w", err)
	}
	defer func() { _ = mod.Close(context.Background()) }()

	allocFn := mod.ExportedFunction("alloc")
	transformFn := mod.ExportedFunction("transform")
	if allocFn == nil || transformFn == nil {
		return nil, fmt.Errorf("plugin missing required 'alloc' or 'transform' exports")
	}

	res, err := allocFn.Call(ctx, uint64(len(input)))
	if err != nil {
		return nil, fmt.Errorf("wasm alloc: %w", err)
	}
	inPtr := uint32(res[0])
	if !mod.Memory().Write(inPtr, input) {
		return nil, fmt.Errorf("wasm memory write out of bounds")
	}

	outRes, err := transformFn.Call(ctx, uint64(inPtr), uint64(len(input)))
	if err != nil {
		return nil, fmt.Errorf("wasm transform (timeout or trap): %w", err)
	}

	outPtr := uint32(outRes[0] >> 32)
	outLen := uint32(outRes[0])
	outBytes, ok := mod.Memory().Read(outPtr, outLen)
	if !ok {
		return nil, fmt.Errorf("wasm memory read out of bounds")
	}

	// Copy out before mod.Close() frees the Wasm linear memory.
	result := make([]byte, len(outBytes))
	copy(result, outBytes)
	return result, nil
}
```

## Architectural Decision Matrix

| Architecture | Crash & Version Isolation | Per-Call Latency | Supported Languages | Best Production Use Case |
|---|---|---|---|---|
| PATH Executables (`kubectl-*`) | Complete (OS process) | ~5–20 ms (`fork`/`exec`) | Any (Go, Rust, Python, Bash) | CLI subcommand extensions invoked once per command |
| `hashicorp/go-plugin` (gRPC) | Complete (long-lived process) | ~30–80 µs (Unix socket RPC) | Any language with gRPC | Heavy stateful providers (`terraform`, `vault`), cloud SDKs |
| `wazero` Wasm (`GOOS=wasip1`) | Complete (sandboxed linear RAM) | ~50–150 ns (JIT call) | Go 1.24+, Rust, Zig, C | High-QPS filters, custom lint rules, untrusted expressions |
| Native `plugin.Open` (`.so`) | None (shared address space) | ~2 ns (direct call) | Exact same Go version + deps | Avoid in public distributions |

Verify that every `PluginHost`, `api.Module`, and `go-plugin` client closes cleanly on error paths with [gohawk](https://gohawk.dev/), and audit plugin dependencies with [govulncheck](https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck).

## Related Chapters

- [244 Plugin Buildmode Pitfalls](../25-go-deep-dives/244-plugin-buildmode-pitfalls.md)
- [234 Wasm and wasip1](../25-go-deep-dives/234-wasm-wasip1.md)
- [323 Command Trees, Zero-Alloc Flag Parsing, and Shell Completion Engines](323-command-trees-kong-cobra-completions.md)
