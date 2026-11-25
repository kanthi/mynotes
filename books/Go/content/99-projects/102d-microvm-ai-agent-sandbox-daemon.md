---
title: '102d. Project: MicroVM / Linux AI Agent Execution Sandbox Daemon'
---

# Project: MicroVM / Linux AI Agent Execution Sandbox Daemon

Untrusted LLM-generated code must never run directly on a shared host. Build a Go sandboxing daemon that launches isolated coding-agent environments using **Firecracker microVMs** (or **Linux namespaces + cgroups v2 + seccomp-bpf** when nested KVM is unavailable), with pause/resume snapshots, injected ephemeral credentials, and strict `nftables`/TAP egress filtering.

## Isolation Architecture

```text
AI Agent Orchestrator (gRPC / Unix Socket)
        |
        v
[ Sandbox Daemon (Go) ]
   ├── 1. Allocate per-sandbox cgroup v2 (cpu.max, memory.max, pids.max, io.max)
   ├── 2. Create isolated network namespace + TAP/veth + nftables egress allowlist
   ├── 3. Boot Firecracker MicroVM (or unshared rootfs + seccomp profile) < 250ms
   ├── 4. Inject short-lived OIDC/API token via tmpfs (never baked into image)
   └── 5. Support Pause/Snapshot -> Resume -> Destroy on TTL or completion
```

## Linux Namespace + `cgroups v2` Sandbox Launcher in Go

```go
package sandbox

import (
	"context"
	"fmt"
	"os"
	"os/exec"
	"path/filepath"
	"syscall"
)

type Limits struct {
	MemoryBytes int64
	MaxPIDs     int
}

func ConfigureCgroupV2(cgroupDir string, lim Limits) error {
	if err := os.MkdirAll(cgroupDir, 0o755); err != nil {
		return fmt.Errorf("mkdir cgroup: %w", err)
	}
	if err := os.WriteFile(filepath.Join(cgroupDir, "memory.max"), []byte(fmt.Sprintf("%d", lim.MemoryBytes)), 0o644); err != nil {
		return fmt.Errorf("set memory.max: %w", err)
	}
	if err := os.WriteFile(filepath.Join(cgroupDir, "pids.max"), []byte(fmt.Sprintf("%d", lim.MaxPIDs)), 0o644); err != nil {
		return fmt.Errorf("set pids.max: %w", err)
	}
	return nil
}

func RunIsolated(ctx context.Context, cmdPath string, args ...string) error {
	cmd := exec.CommandContext(ctx, cmdPath, args...)
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS |
			syscall.CLONE_NEWPID |
			syscall.CLONE_NEWNS |
			syscall.CLONE_NEWNET |
			syscall.CLONE_NEWUSER,
		UidMappings: []syscall.SysProcIDMap{{ContainerID: 0, HostID: os.Getuid(), Size: 1}},
		GidMappings: []syscall.SysProcIDMap{{ContainerID: 0, HostID: os.Getgid(), Size: 1}},
	}
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	return cmd.Run()
}
```

## Security & Acceptance Drills

- **Fork-bomb test:** Run `:(){ :|:& };:` inside the sandbox and verify `pids.max` stops it immediately without affecting the host.
- **Egress test:** Attempt outbound connections to `169.254.169.254` (cloud metadata) and arbitrary IPs; verify `nftables` drops everything except explicitly allowlisted package registries.
- **Snapshot test:** Pause a running Firecracker microVM via its Unix socket API (`PATCH /vm {"state":"Paused"}`), snapshot memory + rootfs, and restore in < 100ms.
