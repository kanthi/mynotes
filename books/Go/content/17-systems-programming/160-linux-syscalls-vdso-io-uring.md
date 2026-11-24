---
title: Linux Syscalls, vDSO Fast Paths, and io_uring vs. netpoll
---

# Linux Syscalls, vDSO Fast Paths, and `io_uring` vs. `netpoll`

Every interaction between Go and the OS kernel crosses the Ring 3 -> Ring 0 privilege boundary. Unlike C, Go cannot blindly execute `SYSCALL` without coordinating with the GMP scheduler and GC—otherwise a blocked thread stalls its `P` and blocks Stop-The-World GC phases.

## `unix.Syscall6` vs. `unix.RawSyscall6`

```text
1. unix.Syscall6 (Blocking-Safe: ~110 ns)
   G1 (_Grunning) -> entersyscall() (_Gsyscall, detaches P for sysmon retake)
                  -> SYSCALL (Ring 3 -> Ring 0)
                  -> exitsyscall() (reacquires P or parks G1)

2. unix.RawSyscall6 (Non-Blocking Fast Path: ~65 ns — NO Scheduler Hook)
   G1 (_Grunning) -> SYSCALL directly (P remains locked! NEVER block!)
```

> [!CAUTION]
> Never call `unix.RawSyscall` on any syscall that can block (disk I/O, pipes, locks): `sysmon` cannot retake `P` and the GC cannot start STW until it returns. Reserve `RawSyscall` strictly for instantaneous in-memory kernel lookups (`getpid`, `gettid`, `epoll_ctl`, `fcntl`). Always pass `unix.O_CLOEXEC` when opening raw FDs.

## Linux vDSO: Zero-Syscall `time.Now()` and `crypto/rand.Read`

During startup (`runtime.vdsoauxv`), Go parses the kernel-mapped `vdso` ELF page directly (no `libc`) to call:
- **`__vdso_clock_gettime` (~12 ns, 0 syscalls):** Reads `RDTSC` scaled by the kernel's `vvar` page.
- **`__vdso_getrandom` (Linux 6.11+ & Go 1.24–1.27):** Runs ChaCha20 CSPRNG in user space for `crypto/rand.Read` and `crypto/rand.Text()`.

## Modern Linux Syscalls Behind Go 1.24–1.27

| Go 1.24–1.27 Feature | Underlying Linux Syscall | Guarantee |
|---|---|---|
| `os.OpenRoot(dir)` (Go 1.24–1.25) | `openat2` (`RESOLVE_BENEATH \| RESOLVE_NO_MAGICLINKS`) | Kernel-enforced atomic traversal defense (immune to TOCTOU) |
| `os.Process.WithHandle` (Go 1.26) | `pidfd_open` & `pidfd_send_signal` | Immutable FD immune to PID reuse races |
| Heap VMA tagging (Go 1.25) | `prctl(PR_SET_VMA_ANON_NAME, "Go: heap")` | Clear RSS attribution in `/proc/self/maps` |

```go
//go:build linux

package kernelsandbox

import (
	"fmt"
	"os"

	"golang.org/x/sys/unix"
)

func OpenBeneath(rootDirFD int, relPath string) (*os.File, error) {
	how := &unix.OpenHow{
		Flags:   unix.O_RDONLY | unix.O_CLOEXEC,
		Resolve: unix.RESOLVE_BENEATH | unix.RESOLVE_NO_MAGICLINKS | unix.RESOLVE_NO_XDEV,
	}
	fd, err := unix.Openat2(rootDirFD, relPath, how)
	if err != nil {
		return nil, fmt.Errorf("openat2 blocked path %q: %w", relPath, err)
	}
	return os.NewFile(uintptr(fd), relPath), nil
}
```

## Why Go Keeps `epoll` for Sockets While Disk Engines Use `io_uring`

- **Sockets (`epoll` readiness model):** Idle goroutines park *before* allocating read buffers—500,000 idle connections don't pin gigabytes of pre-allocated buffers.
- **NVMe Disk (`io_uring` completion model):** Linux `epoll` does not support regular disk files (`EPERM`), so standard `os.File.Read` blocks OS threads (`M`). For high-IOPS `O_DIRECT` storage engines, `io_uring` submits 128 reads via mmap'd `SQ`/`CQ` ring buffers (pinning Go buffers with `runtime.Pinner`) in 1 `io_uring_enter` syscall (or 0 with `SQPOLL`).

Verify raw `int` FD and `runtime.Pinner` cleanup with [gohawk](https://gohawk.dev/) (`go tool gohawk ./...`) and audit syscalls with `strace -f -c ./app`.

## Related Chapters

- [213 Netpoller Network I/O](../25-go-deep-dives/213-netpoller-network-io.md)
- [175b os.Root Traversal-Resistant Files](../22-security-hardening/175b-os-root-traversal-resistant-files.md)
- [161 eBPF in Go, cilium/ebpf, Ringbuf, and Uprobes](161-ebpf-cilium-ringbuf-uprobes.md)
