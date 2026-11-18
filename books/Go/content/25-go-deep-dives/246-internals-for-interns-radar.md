---
title: 'Article Radar: Internals for Interns + Kin'
---

# Article Radar: Internals for Interns + Kin

## Overview

This radar catalogs **diagram-rich Go internals writing** in the same vein as [*Inside Go's synctest*](https://internals-for-interns.com/posts/inside-go-synctest/)—runtime source, compiler pipeline, and systems guests. Use it to prioritize reading and to track what this monorepo has absorbed into local chapters.

## Diagram: map sources → book chapters

```text
  IFI runtime series          this book
  ─────────────────          ─────────
  bootstrap          ──►     248
  allocator          ──►     247
  scheduler          ──►     201, 256
  GC                 ──►     208, 255
  netpoll / sysmon   ──►     213, 239
  synctest           ──►     223
```

## Internals for Interns — Runtime series

| Article | URL | Book landing |
|---------|-----|----------------|
| Bootstrap | [understanding-go-runtime](https://internals-for-interns.com/posts/understanding-go-runtime/) | [248](248-runtime-bootstrap.md) |
| Memory allocator | [go-memory-allocator](https://internals-for-interns.com/posts/go-memory-allocator/) | [247](247-memory-allocator-deep-dive.md) |
| Scheduler | [go-runtime-scheduler](https://internals-for-interns.com/posts/go-runtime-scheduler/) | [201](201-scheduler-gmp-deep-dive.md) |
| Garbage collector | [go-garbage-collector](https://internals-for-interns.com/posts/go-garbage-collector/) | [208](208-gc-internals-pacer.md) |
| System monitor | [go-sysmon](https://internals-for-interns.com/posts/go-sysmon/) | [239](239-sysmon-scavenger-runtime.md) |
| Network poller | [go-netpoller](https://internals-for-interns.com/posts/go-netpoller/) | [213](213-netpoller-network-io.md) |
| Slices, maps, channels | [go-runtime-slices-maps-channels](https://internals-for-interns.com/posts/go-runtime-slices-maps-channels/) | [203](203-channels-select-internals.md), [205](205-slice-map-internals.md) |
| select | [go-runtime-select](https://internals-for-interns.com/posts/go-runtime-select/) | [249](249-selectgo-compiler-runtime.md) |
| Stacktraces | [go-runtime-stacktraces](https://internals-for-interns.com/posts/go-runtime-stacktraces/) | [250](250-stacktraces-runtime-metadata.md) |
| Reflect | [go-runtime-reflect](https://internals-for-interns.com/posts/go-runtime-reflect/) | [211](211-reflect-internals.md) |
| Profiling | [go-runtime-profiling](https://internals-for-interns.com/posts/go-runtime-profiling/) | [222](222-pprof-trace-under-the-hood.md) |
| synctest (guest) | [inside-go-synctest](https://internals-for-interns.com/posts/inside-go-synctest/) | [223](223-testing-synctest-bubbles.md) |
| mmap vs pread (guest) | [mmap-vs-pread-go-storage-engine](https://internals-for-interns.com/posts/mmap-vs-pread-go-storage-engine/) | [251](251-mmap-vs-pread-storage.md) |

Series hub: [Understanding the Go Runtime](https://internals-for-interns.com/series/understanding-the-go-runtime/).

## Internals for Interns — Compiler series

| Phase | URL | Book landing |
|-------|-----|----------------|
| Scanner | [the-go-lexer](https://internals-for-interns.com/posts/the-go-lexer/) | [209](209-compiler-ssa-pgo.md) + radar |
| Parser | [the-go-parser](https://internals-for-interns.com/posts/the-go-parser/) | 209 |
| Type checker | [the-go-type-checker](https://internals-for-interns.com/posts/the-go-type-checker/) | 209 |
| Unified IR | [go-compiler-unified-ir](https://internals-for-interns.com/posts/go-compiler-unified-ir/) | 209 |
| IR | [the-go-ir](https://internals-for-interns.com/posts/the-go-ir/) | 209 |
| SSA | [the-go-ssa](https://internals-for-interns.com/posts/the-go-ssa/) | 209 |
| Machine code | [the-go-machine-code](https://internals-for-interns.com/posts/the-go-machine-code/) | 209 |
| Linker | [the-go-linker](https://internals-for-interns.com/posts/the-go-linker/) | [210](210-link-build-modes-race.md) |
| PGO | [go-compiler-pgo](https://internals-for-interns.com/posts/go-compiler-pgo/) | [186d](../24-performance-engineering/186d-inside-pgo-compiler-internals.md) |

## Kindred sources (same “diagram + source” energy)

| Source | Why |
|--------|-----|
| [antonz.org Gist of Go: Concurrency](https://antonz.org/go-concurrency/) | Interactive curriculum → part [10](../10-concurrency-ground-up/250-concurrency-ground-up-index.md) |
| [iximiuz Labs: Using Go for Systems Programming](https://labs.iximiuz.com/tutorials/go-as-systems-programming-language-a667d22b) | libc vs direct syscalls, static binaries, `CGO_ENABLED=0` → [140a](../17-systems-programming/140a-go-as-systems-language.md) |
| [Go scheduler talk (Vyukov)](https://www.youtube.com/watch?v=-K11rY57K7k) | Original lightweight concurrency design |
| [go.dev blog](https://go.dev/blog/) | Official GC, GOMAXPROCS, experiments |
| Channel runtime posts (e.g. gaborkoos / similar) | hchan latency mental models → [203](203-channels-select-internals.md) |
| Zero-copy sendfile/splice writeups | → [224](224-zero-copy-io.md) |

## How we absorb articles

```text
1. Extract durable models (not copy prose)
2. Add mermaid/ASCII diagrams in-repo
3. Cross-link experiments + GOROOT reading tips
4. Attribute the source series at chapter tops where used heavily
5. Re-check after each Go minor release
```

## Experiment

Pick one unread row in the tables above; after reading, open the matching book chapter and add one personal note (sticky TODO in a `_planning` file if you keep one) for anything the monorepo still under-explains.

**Try next:** On each Go release, re-open the GC + scheduler posts and diff against [208](208-gc-internals-pacer.md) / [201](201-scheduler-gmp-deep-dive.md).
