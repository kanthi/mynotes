---
title: OS Signal Handling, sigaltstack, and Synchronous Trap Internals
---

# OS Signal Handling, `sigaltstack`, and Synchronous Trap Internals

Every running Go process on Linux or macOS constantly processes POSIX signals: `SIGURG` every 10 ms for preemption, `SIGPROF` at 100 Hz for CPU profiling, `SIGSEGV`/`SIGFPE` for nil dereferences and divide-by-zero traps (converted into catchable Go panics), and external `SIGINT`/`SIGTERM`/`SIGHUP`/`SIGWINCH` signals routed to `os/signal.NotifyContext`.

## Why Signals Need `sigaltstack` and `m.gsignal`

A kernel signal frame (`ucontext_t` + `siginfo_t` + AVX2/AVX-512 `fpstate`) consumes **1.5–3 KB**, which would instantly overflow a 2 KB user goroutine stack. During `mcommoninit()`, the runtime allocates a dedicated 32 KB `m.gsignal` goroutine per OS thread `M`, registers it via `sigaltstack(2)`, and installs handlers with `SA_ONSTACK`:

```text
OS Thread M0 Normal Execution                   Kernel Delivers Signal (SIGURG / SIGPROF / SIGSEGV)
+-----------------------------------------+     +-------------------------------------------------+
| m0.curg: Goroutine G12 (2 KB Stack)     |     | 1. Kernel checks SA_ONSTACK flag on sigaction   |
|         (Too small for kernel ucontext) |     | 2. Switches RSP to m0.gsignal (32 KB alt stack) |
+-----------------------------------------+     | 3. Pushes ucontext_t + siginfo_t onto gsignal   |
| m0.gsignal: Dedicated Signal G (32 KB)  |     | 4. Jumps to runtime.sigtramp (asm_amd64.s)      |
|   Registered with kernel via            |     | 5. sigtramp swaps R14 (*g) from G12 -> gsignal  |
|   sigaltstack(2) during mcommoninit()   |     |    and calls runtime.sigtrampgo(sig, info, ctx) |
+-----------------------------------------+     +-------------------------------------------------+
```

## Four Signal Categories in `runtime.sighandler`

| Signal Category | Examples | Runtime Action in `sighandler` |
|---|---|---|
| Runtime preemption | `SIGURG` | Sent by `sysmon` after >10 ms. Verifies safe-point and rewrites `ctxt.rip` to `runtime.asyncPreempt`. |
| CPU profiler sampling | `SIGPROF` | Walks `gp`'s stack lock-free (`sigprof`), reads `gp.labels`, writes to `profBuf`. |
| Synchronous hardware traps | `SIGSEGV`, `SIGBUS`, `SIGFPE` | When `info.si_code != SI_USER` in Go code, `preparePanic` rewrites `ucontext_t.RIP` to `runtime.sigpanic`. |
| Async process signals | `SIGINT`, `SIGTERM`, `SIGHUP`, `SIGWINCH`, `SIGQUIT` | Sets bit in lock-free `sig.mask` (`atomic.Or32`) and wakes `signal_recv` via `sigNoteWakeup(&sig.note)`. |

## Turning Hardware `SIGSEGV` Into a Catchable Go `panic`

```text
Synchronous Hardware Fault -> Go Panic Transformation
1. Goroutine G12 executes: MOVQ $1, 0(AX)  (where AX == 0x0)
2. CPU MMU faults -> Kernel switches RSP to m.gsignal & runs runtime.sighandler
3. sighandler verifies fault occurred in Go code & si_code != SI_USER:
   - Sets G12.sig = _SIGSEGV, G12.sigcode0 = info.si_code, G12.sigpc = ctxt.rip()
4. preparePanic() rewrites the interrupted ucontext_t on the gsignal stack:
   - Pushes old RIP onto G12's stack (preserving exact faulting source line)
   - Overwrites ucontext_t.RIP = abi.FuncPCABIInternal(runtime.sigpanic)
5. sighandler returns -> Kernel sigreturn restores G12 with NEW RIP
6. G12 resumes in user space at runtime.sigpanic() -> calls panicmem()
```

> [!NOTE]
> An external `kill -SEGV <pid>` sets `si_code = SI_USER (0)` rather than `SEGV_MAPERR (1)`, so Go treats it as a fatal asynchronous crash instead of a recoverable panic.

## Lock-Free `os/signal.Notify` Bridge

Inside `sighandler`, the interrupted thread `M` may already hold `hchan.lock` or `sched.lock`, so allocating memory or acquiring any mutex would self-deadlock. Instead, `src/runtime/sigqueue.go` atomically sets a bit in `sig.mask` (`[3]uint32`), wakes `signal_recv` via `futex`, and `os/signal` performs a **non-blocking** `select { case c <- sig: default: }` send—which is why unbuffered signal channels drop signals!

## Complete Signal Lifecycle: Graceful Shutdown, `SIGHUP`, and `SIGUSR1`

```go
//go:build unix

package siglifecycle

import (
	"context"
	"log/slog"
	"os"
	"os/signal"
	"runtime/pprof"
	"sync"
	"syscall"
)

func RunDaemonLifecycle(
	parentCtx context.Context,
	reloadConfig func(ctx context.Context) error,
	runServer func(ctx context.Context) error,
) error {
	ctx, stopShutdown := signal.NotifyContext(parentCtx, os.Interrupt, syscall.SIGTERM)
	defer stopShutdown()

	// Unregister after 1st Ctrl+C so a 2nd Ctrl+C force-exits immediately.
	stopAfterFirst := context.AfterFunc(ctx, func() {
		slog.Warn("received shutdown signal; draining connections (press Ctrl+C again to force exit)")
		stopShutdown()
	})
	defer stopAfterFirst()

	// Always buffer signal channels (capacity >= 1); go vet's sigchanyzer enforces this.
	ctrlCh := make(chan os.Signal, 2)
	signal.Notify(ctrlCh, syscall.SIGHUP, syscall.SIGUSR1)
	defer signal.Stop(ctrlCh)

	var wg sync.WaitGroup
	wg.Go(func() {
		for {
			select {
			case <-ctx.Done():
				return
			case sig := <-ctrlCh:
				switch sig {
				case syscall.SIGHUP:
					if err := reloadConfig(ctx); err != nil {
						slog.Error("config reload failed", "err", err)
					}
				case syscall.SIGUSR1:
					// Dumps all goroutines with Go 1.27 pprof labels without exiting.
					_ = pprof.Lookup("goroutine").WriteTo(os.Stderr, 2)
				}
			}
		}
	})

	err := runServer(ctx)
	stopShutdown()
	wg.Wait()
	return err
}
```

## Production Rules and Tooling

- **Cgo `SA_ONSTACK` requirement:** Any C/C++ library linked via Cgo that installs a `sigaction` handler without `SA_ONSTACK` will execute on a 2 KB Go stack and corrupt memory.
- **Avoid leaking `signal.Notify` registrations:** `signal.Notify` retains `ch` in a global map until `signal.Stop(ch)` is called. Verify `signal.Stop` and watcher goroutine joins with [gohawk](https://gohawk.dev/) (`go tool gohawk ./...`) and catch unbuffered channels with `go vet` (`sigchanyzer`).
- **Use `kill -QUIT <pid>` before `kill -9`:** `SIGQUIT` dumps every goroutine stack (with Go 1.27 `pprof.Labels`) and CPU registers to stderr before exiting. For `mmap` `SIGBUS` faults, use `debug.SetPanicOnFault(true)`.

## Related Chapters

- [201 Scheduler GMP Deep Dive](201-scheduler-gmp-deep-dive.md)
- [228 Cgo Scheduler Transitions](228-cgo-scheduler-transitions.md)
- [308 Signals and Context](../09-cli-tools-in-go/308-signals-context-cli.md)
