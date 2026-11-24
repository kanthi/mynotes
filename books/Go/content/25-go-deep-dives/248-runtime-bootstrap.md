---
title: Runtime Bootstrap, rt0_go, schedinit, and Package init DAG
---

# Runtime Bootstrap, `rt0_go`, `schedinit`, and Package `init` DAG

Before a single line of `main.main` runs, the OS jumps into Go's runtime bootstrap sequence. The runtime probes CPU vector extensions (AVX2, AVX-512, AES-NI), links the Linux vDSO clock and CSPRNG pages, initializes the root system goroutine (`runtime.g0`) and root OS thread (`runtime.m0`), reserves virtual memory arenas with a randomized 64-bit base address (`mallocinit`), sizes the GMP processor array (`allp`) to the container's cgroup v2 `cpu.max` quota, enables the Green Tea Garbage Collector, and walks the topological dependency graph of every imported package's `init()` functions.

## The Four-Phase Bootstrap Pipeline

```text
OS Kernel (execve) -> Jumps to ELF e_entry: _rt0_amd64_linux(SB)
                                 |
                                 v
+-------------------------------------------------------------------------------+
| Phase 1: Assembly Bootstrap (runtime.rt0_go in src/runtime/asm_amd64.s)       |
| - Initializes root goroutine (runtime.g0) & root OS thread (runtime.m0)       |
| - Runs CPUID hardware feature detection (AVX2, AVX-512, AES-NI, BMI2)         |
| - Initializes Thread-Local Storage (TLS) & pins R14 = &runtime.g0             |
+---------------------------------------+---------------------------------------+
                                        |
                                        v
+-------------------------------------------------------------------------------+
| Phase 2: Runtime Subsystem Init (osinit & schedinit in src/runtime/proc.go)   |
| - runtime.args(): Parses ELF auxv & resolves Linux vDSO symbols               |
| - runtime.osinit(): Reads CPU count & cgroup v2 cpu.max (Go 1.25–1.27)        |
| - runtime.schedinit(): Initializes stack pools, mheap (randomized 64-bit      |
|   base in Go 1.26+), SwissTable hash seeds, itabTable, and allp[GOMAXPROCS]   |
+---------------------------------------+---------------------------------------+
                                        |
                                        v
+-------------------------------------------------------------------------------+
| Phase 3: Spawning G1 & Starting the GMP Scheduler (newproc & mstart)          |
| - runtime.newproc(&mainPC): Allocates Goroutine #1 (G1) pointing to           |
|   runtime.main (NOT user main.main) and places G1 on P0's local run queue     |
| - runtime.mstart(): Enters runtime.schedule() on M0 -> switches to G1         |
+---------------------------------------+---------------------------------------+
                                        |
                                        v
+-------------------------------------------------------------------------------+
| Phase 4: runtime.main() on Goroutine G1 (src/runtime/proc.go)                 |
| 1. Spawns sysmon background watchdog thread on a dedicated M                  |
| 2. Executes runtime..inittask (internal runtime package init() functions)     |
| 3. Calls gcenable() -> starts Green Tea GC mark workers & memory scavenger    |
| 4. Executes main..inittask -> walks topological DAG of user package init()s   |
| 5. Calls main.main() (user entry point)                                       |
| 6. Calls exit(0) immediately when main.main() returns                         |
+-------------------------------------------------------------------------------+
```

## Phase 1: `runtime.rt0_go` (`src/runtime/asm_amd64.s`)

When the Linux ELF loader maps the binary, `RSP` points to `argc`, `argv`, `envp`, and the ELF auxiliary vector (`auxv`):

```text
TEXT _rt0_amd64_linux(SB),NOSPLIT,$-8
    JMP  _rt0_amd64(SB)

TEXT _rt0_amd64(SB),NOSPLIT,$-8
    MOVQ 0(SP), DI          // DI = argc
    LEAQ 8(SP), SI          // SI = argv
    JMP  runtime·rt0_go(SB)
```

Inside `runtime.rt0_go`:

1. **Sets `g0` stack bounds:** Configures `g0.stack.hi = SP` and `g0.stack.lo = SP - 64*1024` on the initial OS thread (`runtime.m0`).
2. **Probes CPU features (`CPUID` / `HWCAP`):** Populates `internal/cpu.X86` to decide whether `memmove` uses AVX2/ERMS, SwissTable hashing uses AESENC, and the Go 1.26–1.27 Green Tea GC uses AVX2/AVX-512 SIMD bitmap scanning.
3. **Binds `m0` and `g0`:** Links `m0.g0 = &g0`, `g0.m = &m0`, and pins `&g0` in hardware register `R14` (amd64) or `R28` (arm64).
4. **Calls `check`, `args`, `osinit`, `schedinit`, `newproc(mainPC)`, and `mstart` (never returns).**

## Phase 2: `runtime.schedinit()` (`src/runtime/proc.go`)

| Subsystem Call | Initialization Responsibility |
|---|---|
| `stackinit()` | Initializes 2 KB–32 KB stack span free-lists |
| `mallocinit()` | Reserves 64 MB `heapArena` virtual space with Go 1.26+ randomized 64-bit base |
| `alginit()` / `fastrandinit()` | Seeds AES/wyhash SwissTable map hashes and per-M ChaCha8 PRNGs |
| `mcommoninit(gp.m, -1)` | Assigns `M0` ID and allocates its 32 KB `gsignal` stack (`sigaltstack`) |
| `modulesinit()` / `itabsinit()` | Loads `.gopclntab` module descriptors and global lock-free `itabTable` |
| `gcinit()` | Configures Green Tea GC pacer and `GOMEMLIMIT` ceiling |
| `defaultGOMAXPROCS(ncpu)` | Inspects both logical CPUs and Linux cgroup v2 `cpu.max` (Go 1.25–1.27), then calls `procresize(procs)` |

## Phase 4: Package `init()` Topological DAG and Cold-Start Latency

All package `init()` functions run sequentially on Goroutine #1 (`G1`) in topological depth-first dependency order via compiler-generated `runtime.initTask` descriptors before `main.main` starts.

> [!WARNING]
> Every imported package's `init()` runs unconditionally—even for `my-cli --version` or `go test`. Heavy regex compilation, schema parsing, or I/O in `init()` penalizes every cold start. Replace eager package-level initializers with `sync.OnceValue`:

```go
package configschema

import (
	"regexp"
	"sync"
)

// Compiled lazily in a thread-safe sync.Once wrapper ONLY on first call.
var RouteRegex = sync.OnceValue(func() *regexp.Regexp {
	return regexp.MustCompile(`^/api/v[0-9]+/tenants/([a-z0-9-]+)$`)
})
```

## Profiling Startup Latency and Stepping Through Bootstrap

Measure exact wall time, heap bytes, and allocations per package `init()` without code changes:

```bash
GODEBUG=inittrace=1 ./my-service --version
```

```text
init internal/bytealg @0.008 ms, 0 ms clock, 0 bytes, 0 allocs
init runtime @0.042 ms, 0.031 ms clock, 0 bytes, 0 allocs
init crypto/tls @1.42 ms, 0.85 ms clock, 48256 bytes, 412 allocs
init main @20.8 ms, 0.12 ms clock, 1024 bytes, 8 allocs
```

Step through pre-`main` bootstrap live in [Delve](https://github.com/go-delve/delve):

```bash
dlv exec ./my-service
# (dlv) break _rt0_amd64_linux
# (dlv) break runtime.schedinit
# (dlv) break runtime.main
# (dlv) continue
```

> [!IMPORTANT]
> When `main.main()` returns, `runtime.main()` calls `exit(0)` immediately without waiting for background goroutines. Ban goroutine spawning inside `func init()` and run [gohawk](https://gohawk.dev/) (`go tool gohawk ./...`) to verify `goroutineownership` and `resourcelifetime` across startup and shutdown paths.

## Related Chapters

- [201 Scheduler GMP Deep Dive](201-scheduler-gmp-deep-dive.md)
- [232 Build Cache, Action DAG, and Linker Internals](232-build-cache-reproducibility.md)
- [258 OS Signals and Runtime](258-os-signals-runtime.md)
