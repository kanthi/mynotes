---
title: Linux Namespaces, Cgroups v2, pivot_root, and Container Runtimes in Go
---

# Linux Namespaces, Cgroups v2, `pivot_root`, and Container Runtimes in Go

OCI runtimes (`runc`, `containerd`) build Linux containers from three kernel primitives: **Namespaces** (`Cloneflags` / `setns`), **Cgroups v2** (`memory.max`, `cpu.max`, `pids.max`), and **Filesystem Isolation** (`pivot_root` + `os.Root`). Writing container runtimes in Go requires respecting how Go's multi-threaded GMP scheduler interacts with Linux per-thread namespaces.

## The Multi-Threaded Go Namespace Trap (`runtime.LockOSThread`)

```text
The Unpinned Goroutine Namespace Corruption Bug
1. Goroutine G7 (on OS Thread M2) calls unix.Setns(containerNetFD) -> only M2 switches netns
2. G7 parks on netpoll -> resumes later on M1 (still in HOST netns!)
3. Unrelated HTTP handler G9 schedules onto M2 -> trapped inside container netns!
```

1. **Always call `runtime.LockOSThread()`** before `unix.Setns`.
2. **Destroy tainted threads on restore failure:** If restoring the original namespace fails, return *without* calling `runtime.UnlockOSThread()`—the Go runtime terminates that OS thread (`mexit`) instead of returning it to the GMP pool.
3. **Use self re-exec (`/proc/self/exe`) for `CLONE_NEWNS` / `CLONE_NEWUSER`:** Linux forbids `setns`/`unshare` of mount/user namespaces from multi-threaded processes (`EINVAL`); spawn `/proc/self/exe child-init` with `SysProcAttr.Cloneflags` so namespaces are created during `clone()` before Go spawns threads.

## Safe Network Namespace Execution and Cgroups v2 via `os.Root`

```go
//go:build linux

package containerops

import (
	"fmt"
	"os"
	"runtime"
	"strconv"

	"golang.org/x/sys/unix"
)

func DoInNetNS(targetNetNSFD int, fn func() error) error {
	done := make(chan error, 1)
	go func() {
		runtime.LockOSThread()
		origFD, err := unix.Open(fmt.Sprintf("/proc/self/task/%d/ns/net", unix.Gettid()), unix.O_RDONLY|unix.O_CLOEXEC, 0)
		if err != nil {
			done <- err
			return // Stay locked -> runtime destroys M.
		}
		defer unix.Close(origFD)

		if err := unix.Setns(targetNetNSFD, unix.CLONE_NEWNET); err != nil {
			done <- err
			return
		}
		fnErr := fn()
		if err := unix.Setns(origFD, unix.CLONE_NEWNET); err != nil {
			done <- fmt.Errorf("restore host netns (destroying OS thread): %w", err)
			return
		}
		runtime.UnlockOSThread()
		done <- fnErr
	}()
	return <-done
}

type Limits struct {
	MemoryMaxBytes int64
	CPUQuotaMicros int64 // e.g. 200000 out of 100000 = 2.0 cores
	PidsMax        int64
}

// ConfigureCgroupV2 writes memory.max, cpu.max, and pids.max safely via Go 1.25+ os.Root.
// Go 1.25–1.27 binaries inside this cgroup automatically read cpu.max for GOMAXPROCS.
func ConfigureCgroupV2(cgroupName string, pid int, lim Limits) error {
	cgRoot, err := os.OpenRoot("/sys/fs/cgroup")
	if err != nil {
		return err
	}
	defer cgRoot.Close()
	if err := cgRoot.MkdirAll(cgroupName, 0755); err != nil {
		return err
	}
	sub, err := cgRoot.OpenRoot(cgroupName)
	if err != nil {
		return err
	}
	defer sub.Close()

	if err := sub.WriteFile("memory.max", []byte(strconv.FormatInt(lim.MemoryMaxBytes, 10)), 0644); err != nil {
		return err
	}
	if err := sub.WriteFile("cpu.max", []byte(fmt.Sprintf("%d 100000", lim.CPUQuotaMicros)), 0644); err != nil {
		return err
	}
	if err := sub.WriteFile("pids.max", []byte(strconv.FormatInt(lim.PidsMax, 10)), 0644); err != nil {
		return err
	}
	return sub.WriteFile("cgroup.procs", []byte(strconv.Itoa(pid)), 0644)
}
```

> [!IMPORTANT]
> Prefer `unix.PivotRoot(newRoot, putOld)` + `unix.Unmount(putOld, unix.MNT_DETACH)` over `chroot`, keep the spawning goroutine pinned with `runtime.LockOSThread()` whenever setting `SysProcAttr.Pdeathsig: unix.SIGKILL` (Linux delivers `Pdeathsig` when the spawning OS thread exits!), and verify namespace FD cleanup with [gohawk](https://gohawk.dev/).

## Related Chapters

- [154b Rootless Namespaces](154b-rootless-namespaces.md)
- [243 GOMAXPROCS and Cgroup Containers](../25-go-deep-dives/243-gomaxprocs-cgroup-containers.md)
- [160 Linux Syscalls, vDSO, and io_uring](160-linux-syscalls-vdso-io-uring.md)
