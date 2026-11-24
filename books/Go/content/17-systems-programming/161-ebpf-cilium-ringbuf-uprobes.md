---
title: eBPF in Go, cilium/ebpf, Kernel Ring Buffers, and Go Uprobe Tracing
---

# eBPF in Go, `cilium/ebpf`, Kernel Ring Buffers, and Go Uprobe Tracing

Cloud-native CNI data planes ([Cilium](https://cilium.io/)), zero-instrumentation L7 tracers (OpenTelemetry eBPF / Beyla), and runtime security engines (Tetragon) pair restricted-C eBPF kernel programs with pure-Go (`CGO_ENABLED=0`) control planes built on [github.com/cilium/ebpf](https://github.com/cilium/ebpf) and `bpf2go`.

## Pure-Go CO-RE Architecture

```text
Build Time (bpf2go + clang -target bpf):
  tracer.bpf.c ===(clang -O2 -g -target bpf)===> tracer_bpfel.o (ELF + BTF)
                                                        | (//go:embed)
                                                        v
Runtime (Pure Go, CGO_ENABLED=0):
  1. Reads host kernel types from /sys/kernel/btf/vmlinux
  2. Applies BTF CO-RE struct offset relocations in Go user space
  3. Loads via bpf(BPF_MAP_CREATE) & bpf(BPF_PROG_LOAD) -> Kernel Verifier -> JIT
```

## `BPF_MAP_TYPE_RINGBUF` vs. Legacy Perf Arrays

| Mechanism | Layout | Ordering | Go Consumer |
|---|---|---|---|
| `BPF_MAP_TYPE_PERF_EVENT_ARRAY` | Per-CPU ring buffers (`N` FDs) | Out-of-order across CPUs | Higher polling & copy overhead |
| `BPF_MAP_TYPE_RINGBUF` (Linux 5.8+) | Single shared MPSC mmap ring | Strictly ordered | Zero-alloc `ringbuf.Reader.ReadInto(&rec)` |

## Tracing Go Binaries with Uprobes: Three Runtime Traps

1. **Go `ABIInternal` registers (not C System V ABI):** On amd64, Go 1.17–1.27 passes arguments in `RAX, RBX, RCX, RDI, RSI, R8–R11`—for `(*tls.Conn).Write(b []byte)`, `RAX = c`, `RBX = b.Data`, `RCX = b.Len`, `RDI = b.Cap`.
2. **Extracting `goid` from `R14` / `R28`:** `R14` (amd64) and `R28` (arm64) always hold `*runtime.g`; read `goid` at its DWARF5-discovered offset (`ebpf.CollectionSpec.Variables`) via `bpf_probe_read_user`.
3. **Never attach kernel `uretprobe` to Go functions:** `uretprobe` overwrites the return address on the 2 KB goroutine stack; if `runtime.morestack` / `copystack` relocates the stack before returning, the process crashes. Disassemble the function (`debug/elf` + `x86asm`) and attach standard `uprobe`s to every `RET` instruction offset instead.

## Pure-Go Uprobe Loader and Zero-Alloc Ringbuf Consumer

```go
//go:build linux

package ebpfagent

import (
	"context"
	"encoding/binary"
	"errors"
	"fmt"
	"log/slog"
	"sync"

	"github.com/cilium/ebpf"
	"github.com/cilium/ebpf/link"
	"github.com/cilium/ebpf/ringbuf"
	"github.com/cilium/ebpf/rlimit"
)

func RunTLSUprobeAgent(ctx context.Context, binaryPath string, spec *ebpf.CollectionSpec) error {
	if err := rlimit.RemoveMemlock(); err != nil {
		return fmt.Errorf("remove memlock: %w", err)
	}
	coll, err := ebpf.NewCollection(spec)
	if err != nil {
		return fmt.Errorf("load ebpf collection: %w", err)
	}
	defer coll.Close()

	ex, err := link.OpenExecutable(binaryPath)
	if err != nil {
		return fmt.Errorf("open executable: %w", err)
	}
	up, err := ex.Uprobe("crypto/tls.(*Conn).Write", coll.Programs["trace_go_tls_write"], nil)
	if err != nil {
		return fmt.Errorf("attach uprobe: %w", err)
	}
	defer up.Close()

	rd, err := ringbuf.NewReader(coll.Maps["events"])
	if err != nil {
		return fmt.Errorf("open ringbuf reader: %w", err)
	}
	stopCancel := context.AfterFunc(ctx, func() { _ = rd.Close() })
	defer stopCancel()
	defer rd.Close()

	var wg sync.WaitGroup
	wg.Go(func() {
		var rec ringbuf.Record // Reused across events -> 0 heap allocations.
		for {
			if err := rd.ReadInto(&rec); err != nil {
				if errors.Is(err, ringbuf.ErrClosed) {
					return
				}
				continue
			}
			if len(rec.RawSample) >= 20 {
				pid := binary.LittleEndian.Uint32(rec.RawSample[0:4])
				goid := binary.LittleEndian.Uint64(rec.RawSample[8:16])
				dataLen := binary.LittleEndian.Uint32(rec.RawSample[16:20])
				slog.Info("intercepted Go TLS write", "pid", pid, "goid", goid, "bytes", dataLen)
			}
		}
	})
	wg.Wait()
	return nil
}
```

Inspect JIT-compiled programs with `sudo bpftool prog dump jited id <ID>`, and run [gohawk](https://gohawk.dev/) (`go tool gohawk ./...`) to ensure every `ebpf.Collection`, `link.Link`, and `ringbuf.Reader` closes on all error paths.

## Related Chapters

- [100g eBPF Tracepoint Basics](../99-projects/100g-ebpf-tracepoint-basics.md)
- [100h eBPF XDP Packet Counter](../99-projects/100h-ebpf-xdp-packet-counter.md)
- [230 ABI, Assembly, and nosplit](../25-go-deep-dives/230-abi-assembly-nosplit.md)
