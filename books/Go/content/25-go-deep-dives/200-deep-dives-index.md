---
title: Go Deep Dives Overview
---

# Go Deep Dives Overview


## Diagram: Deep-dive stack

```text
flow:
  [Comp]
       |
       v
  [RT]

  [RT]
       |
       v
  [OS]
```

This part is for readers who already write idiomatic Go and want the **runtime, toolchain, and production mental models** behind real symptoms: latency cliffs, leaks, interface nil traps, timer churn, cgo taxes, and "why is this slow in the pod?"

Earlier parts teach *how to use* the language. These chapters teach *how the machine runs it* — deep enough to debug, not a full fork of `src/runtime`.

## Who This Part Is For

- You have shipped Go services or tools
- You can read basic `pprof` / `trace` output
- You want **code → runtime structures → observables**

If you are still learning syntax, finish Core, Memory, and Concurrency first.

## Learning Paths

### A — Runtime core
201 GMP → 202 memory model → 203 channels → 213 netpoll → 214 timers → 227 mutex → 239 sysmon

### B — Data representation
204 interfaces → 205 slice/map → 218 Swiss maps → 235 strings/bytes → 217 weak/unique → 236 errors

### C — Execution control
206 defer/panic → 215 stacks → 226 context → 220 leaks → 223 synctest

### D — Performance & GC
208 GC → 219 pool/zero-alloc → 222 pprof/trace → 224 zero-copy → 209 compiler/PGO → 243 GOMAXPROCS/cgroup

### E — Toolchain & platforms
210 link/race → 228 cgo → 230 ABI → 231 coverage/fuzz → 232 build cache → 233 experiments → 234 Wasm → 244 plugins

### F — Production craft
225 HTTP transport → 241 TLS → 242 slog handlers → 237 mistakes → 238 debugging → 240 reading source → 245 topic radar

### G — Go Internals: Source Code to Assembly & Runtime (8-Module Track)

| Module | Topics | Chapters |
|--------|--------|----------|
| **01. Go Thinks Simple Works Better** | What makes Go special, the Go way, design decisions, how Go runs code | [01 Why Go Exists](../01-foundations/01-why-go-exists.md), [01b Boring Philosophy](../01-foundations/01b-boring-philosophy.md), [248 Runtime Bootstrap](248-runtime-bootstrap.md) |
| **02. Basic Data Types, Variables & Constants** | Integer bit-widths, IEEE-754 floats, 1-byte bool, pointers, vars/consts, `unsafe.Pointer`, addressable vs. unaddressable | [09b Machine Representation & Addressability](../02-core/09b-machine-representation-addressability.md), [23 Pointers](../03-memory/23-pointers.md), [02 Unsafe Code](../26-long-term-maintenance/02-unsafe-code.md) |
| **03. Arrays, Slices, Strings & Maps** | Fixed-size arrays, slice headers, string immutability, Swiss map internals | [15–17 Core Data Structures](../02-core/15-arrays-slices.md), [205 Slice & Map Internals](205-slice-map-internals.md), [218 Swiss Maps](218-swiss-maps-maphash.md), [235 String/Byte Internals](235-string-byte-internals.md) |
| **04. Structs, Generics & Interfaces** | Struct layout/alignment, `eface`/`iface` runtime mechanics, generic stencils & dictionaries | [25c Alignment](../03-memory/25c-escape-analysis-alignment.md), [204 Interface Representation](204-interface-representation.md), [207 Generics Implementation](207-generics-implementation.md) |
| **05. How Does Go Code Turn into Assembly?** | Stages 1–7: AST, type checking, IR, optimizations, walk, SSA, arch assembly | [272 Compilation Pipeline Hands-On](272-go-compilation-pipeline-hands-on.md), [252 Compiler Frontend](252-compiler-frontend.md), [253 IR/SSA Backend](253-compiler-ir-ssa-backend.md), [230 ABI & Assembly](230-abi-assembly-nosplit.md) |
| **06. Functionality** | `defer`, `panic`/`recover`, dead code elimination, inlining, devirtualization | [206 Defer, Panic & Stack](206-defer-panic-stack.md), [271 Walk, DCE & Devirtualization](271-compiler-walk-dce-devirtualization.md) |
| **07. Memory** | Stack growth/shrinking, escape analysis, heap allocator, GC | [215 Stack Growth](215-stack-growth-copying.md), [25c Escape Analysis](../03-memory/25c-escape-analysis-alignment.md), [247 Memory Allocator](247-memory-allocator-deep-dive.md), [208 GC Internals](208-gc-internals-pacer.md) |
| **08. Concurrency** | Channels, `select`, timers, startup, goroutine lifecycle, scheduler, I/O, netpoller, `sysmon` | [203 Channels & Select](203-channels-select-internals.md), [249 `selectgo`](249-selectgo-compiler-runtime.md), [214 Timers](214-timers-time-after-pitfalls.md), [201 GMP Scheduler](201-scheduler-gmp-deep-dive.md), [263 Goroutine Switch](263-goroutine-switch-mechanics.md), [213 Netpoller](213-netpoller-network-io.md), [239 Sysmon](239-sysmon-scavenger-runtime.md) |

## Full Chapter Map

| # | Topic |
|---|--------|
| 201 | Scheduler (GMP) |
| 202 | Memory model / happens-before |
| 203 | Channels & select |
| 204 | Interfaces (eface/iface) |
| 205 | Slice & map headers |
| 206 | Defer, panic, unwind |
| 207 | Generics implementation |
| 208 | GC internals & pacer |
| 209 | Compiler, SSA, PGO |
| 210 | Link, buildmodes, race |
| 211 | Reflect cost |
| 212 | Modules, MVS, toolchain |
| 213 | Netpoller |
| 214 | Timers / `time.After` |
| 215 | Stack growth |
| 216 | Finalizers & cleanup |
| 217 | `weak` & `unique` |
| 218 | Swiss maps & maphash |
| 219 | `sync.Pool` & zero-alloc |
| 220 | Goroutine leaks |
| 221 | GODEBUG knobs |
| 222 | pprof & tracer internals |
| 223 | testing/synctest (bubbles, durable blocks, fake time) |
| 224 | Zero-copy I/O |
| 225 | HTTP Transport pool |
| 226 | Context cancel trees |
| 227 | Mutex / semaphore runtime |
| 228 | cgo + Go 1.26 scheduler |
| 229 | Iterators / range-over-func |
| 230 | ABI & assembly |
| 231 | Coverage & fuzz engine |
| 232 | Build cache / reproducibility |
| 233 | GOEXPERIMENT catalog |
| 234 | Wasm / wasip1 |
| 235 | String / []byte internals |
| 236 | Error interface cost |
| 237 | Production mistakes map |
| 238 | Production debugging |
| 239 | sysmon & scavenger |
| 240 | Reading runtime source |
| 241 | TLS handshake path |
| 242 | slog handler design |
| 243 | GOMAXPROCS & cgroups |
| 244 | Plugin buildmode pitfalls |
| 245 | X/ecosystem topic radar |
| 246 | Internals-for-Interns article radar |
| 247 | Memory allocator (mcache/mcentral/mheap) |
| 248 | Runtime bootstrap |
| 249 | select + selectgo |
| 250 | Stacktraces + binary metadata |
| 251 | mmap vs pread storage I/O |
| 252 | Compiler frontend (scan/parse/types) |
| 253 | Compiler IR / SSA / link |
| 254 | Race detector internals |
| 255 | Write barriers & mark assist |
| 256 | Work stealing / findrunnable |
| 257 | Type assert / convert runtime |
| 258 | OS signals & runtime |
| 259 | HTTP/2 in Go |
| 260 | database/sql pool |
| 261 | JSON performance |
| 262 | errgroup patterns |
| 263 | Goroutine switch mechanics |
| 264 | Measuring a "compiler will optimize it" idiom (`netip.AddrFrom16(ip.As16())`) |
| 265 | Why Go binaries are big: sections, reachability, linker method rules |
| 266 | [epoll inside the Go scheduler](266-epoll-inside-the-go-scheduler.md) |
| 267 | [GOMAXPROCS and P counts](267-gomaxprocs-and-p-counts.md) |
| 268 | [`go list -json` package metadata](268-go-list-json-package-metadata.md) |
| 269 | [`goroutineleak` pprof profile](269-goroutineleak-pprof.md) |
| 270 | [Open-source architecture teardown: Kubernetes, `etcd`, and Prometheus in Go 1.27](270-oss-architecture-k8s-etcd-prometheus.md) |
| 271 | [Compiler walk pass, dead code elimination, and devirtualization](271-compiler-walk-dce-devirtualization.md) |
| 272 | [Inspecting the Go compilation pipeline: source to assembly](272-go-compilation-pipeline-hands-on.md) |

## Stack Diagram

```text
your code
    |
    v
compiler (types, SSA, inlining, escape, PGO)
    |
    v
runtime (scheduler, poller, timers, maps, GC, defer, cgo)
    |
    v
OS threads + cgroups + network stack + TLS
```

## Diagrams in this part

Prefer **ASCII diagrams** in fenced `text` blocks (works in HTML, PDF, and EPUB). Flagship topics also ship optional **SVG** under `content/25-go-deep-dives/assets/` for HTML. Mermaid is not required for this part.

## Study Method

```text
hypothesis -> minimal repro -> measure (pprof/trace/race) -> model -> fix ownership/allocs/timeouts
```

1. One chapter, one question.
2. Run the experiment; change one knob.
3. Confirm with tools, not vibes.

## Relationship to Other Parts

| Concern | Prefer |
|---------|--------|
| Concurrency patterns | Part 10/11 + 201/203/220 |
| GC knobs tutorial | Part 03/24 + 208/239 |
| Profiling workflow | Part 23/24 + 222/238 |
| Stdlib how-to | Part 07 |
| HTTP resilience | Part 13 + 225/241 |
| Security TLS ops | Part 22 + 241 |

## Honesty Boundary

Runtime details evolve (scheduler, GC, maps, experiments). Chapters teach **stable models** and version-labeled notes (e.g. Go 1.26 Green Tea GC, P-syscall simplification; Go 1.27 size-specialized malloc and `goroutineleak`). Pseudocode is a map for reading source — not a bit-identical clone of `src/runtime`.

## Checklist: Deep-Dive Literacy

- [ ] Draw G/M/P and explain work stealing
- [ ] State happens-before for unlock→lock and send→receive
- [ ] Explain typed-nil interface trap
- [ ] Fix a goroutine leak with context
- [ ] Replace hot `time.After` with timer reuse
- [ ] Tune or justify GOMAXPROCS under cgroup limits
- [ ] Read one CPU + one goroutine profile
- [ ] Find a panic string in `GOROOT/src/runtime`
- [ ] Know when cgo/plugins/Wasm are the wrong tool

When most boxes are automatic, you can track Go releases without fear.
