---
title: CLI Packaging, Self-Updating Binaries, and Cross-Platform Distribution
---

# CLI Packaging, Self-Updating Binaries, and Cross-Platform Distribution

Shipping a production CLI (`kubectl`, `gh`, `docker`, `flyctl`, `tailscale`) requires more than flag parsing and TUI polish. Once compiled, your binary should:

1. **Embed verifiable build provenance:** Expose its Git commit, build timestamp, compiler architecture (`GOAMD64`), and dependency hashes via `.go.buildinfo` (`runtime/debug.ReadBuildInfo`)—even when installed via `go install pkg@latest` without a `Makefile`.
2. **Compile reproducibly:** Produce bit-for-bit identical SHA-256 binaries across build machines (`CGO_ENABLED=0 -trimpath`).
3. **Self-update atomically:** Verify cryptographic release signatures (Ed25519 or Go 1.27's post-quantum `crypto/mldsa` FIPS 204) and swap its own running executable on disk without triggering Linux `ETXTBSY`, cross-mount `EXDEV`, or Windows file-locking failures.

## `.go.buildinfo`: `runtime/debug.ReadBuildInfo` vs. `-ldflags="-X"`

Linker overrides (`go build -ldflags="-s -w -X main.version=v1.4.2"`) overwrite package-level `string` symbols in `.rodata`, but remain empty when users install via `go install`. By default (`-buildvcs=true`), `cmd/go` embeds a structured `.go.buildinfo` section that `debug.ReadBuildInfo()` reads at runtime:

```go
package versioninfo

import "runtime/debug"

var (
	ldVersion = ""
	ldCommit  = ""
	ldDate    = ""
)

type BuildMetadata struct {
	Version    string `json:"version"`
	GoVersion  string `json:"go_version"`
	Commit     string `json:"commit"`
	CommitTime string `json:"commit_time"`
	Dirty      bool   `json:"dirty"`
	OSArch     string `json:"os_arch"`
	GOAMD64    string `json:"goamd64,omitzero"`
}

// Resolve combines -ldflags="-X" overrides with automatic .go.buildinfo VCS fallbacks.
func Resolve() BuildMetadata {
	meta := BuildMetadata{
		Version:    ldVersion,
		Commit:     ldCommit,
		CommitTime: ldDate,
	}

	info, ok := debug.ReadBuildInfo()
	if !ok {
		return meta
	}

	meta.GoVersion = info.GoVersion
	if meta.Version == "" && info.Main.Version != "" && info.Main.Version != "(devel)" {
		meta.Version = info.Main.Version
	}
	if meta.Version == "" {
		meta.Version = "devel"
	}

	var goos, goarch string
	for _, s := range info.Settings {
		switch s.Key {
		case "vcs.revision":
			if meta.Commit == "" {
				meta.Commit = s.Value
			}
		case "vcs.time":
			if meta.CommitTime == "" {
				meta.CommitTime = s.Value
			}
		case "vcs.modified":
			meta.Dirty = (s.Value == "true")
		case "GOOS":
			goos = s.Value
		case "GOARCH":
			goarch = s.Value
		case "GOAMD64":
			meta.GOAMD64 = s.Value
		}
	}
	meta.OSArch = goos + "/" + goarch
	return meta
}
```

## Bit-for-Bit Reproducible Static CLI Builds

```bash
CGO_ENABLED=0 \
GOOS=linux GOARCH=amd64 GOAMD64=v3 \
go build \
  -trimpath \
  -buildvcs=true \
  -ldflags="-s -w -buildid=" \
  -o dist/mycli ./cmd/mycli
```

| Build Flag | Why It Is Required for Reproducible Production CLIs |
|---|---|
| `CGO_ENABLED=0` | Links a 100% static pure-Go binary with zero host `glibc` / `musl` dependencies. |
| `-trimpath` | Strips local filesystem paths from `.gopclntab` stack traces, replacing them with module-relative paths. |
| `-ldflags="-s -w"` | Omits the OS symbol table (`-s`) and DWARF5 debug sections (`-w`), shrinking binary size ~28% while keeping `.gopclntab` and `.go.buildinfo` intact. |
| `-ldflags="-buildid="` | Clears non-deterministic toolchain wrapper build IDs so content hashes depend strictly on source and compiler version. |

## Atomic Self-Updating: Defeating `ETXTBSY`, `EXDEV`, and Windows Locks

```text
1. WHY NAIVE OVERWRITE FAILS ON LINUX (ETXTBSY: "text file busy"):
   os.OpenFile("/usr/local/bin/mycli", os.O_WRONLY|os.O_TRUNC, 0755)
   -> Kernel checks inode's vma->vm_flags: sees VM_DENYWRITE (mapped as running code)
   -> Returns ETXTBSY. You CANNOT write into a running executable's inode on Linux.

2. THE ATOMIC POSIX RENAME PROTOCOL (Linux / macOS / BSD):
+-------------------------------------------------------------------------------+
| Directory: /usr/local/bin (Opened via os.OpenRoot("/usr/local/bin"))          |
|                                                                               |
| Step A: Running "mycli" points to Inode #1042 (mapped in kernel RAM)          |
| Step B: Stream & verify new release into sibling ".mycli.tmp.9f2a" (Inode #88)|
|         inside the EXACT SAME directory (prevents cross-mount EXDEV error)    |
| Step C: fsync(Inode #88) + chmod(0755)                                        |
| Step D: Atomic renameat2(".mycli.tmp.9f2a" -> "mycli")                        |
|         - Directory entry "mycli" now points to Inode #88                     |
|         - Running process continues executing unlinked Inode #1042 cleanly    |
+-------------------------------------------------------------------------------+
```

> [!IMPORTANT]
> **Why staging in `/tmp` fails (`EXDEV`):** On Linux servers, `/tmp` is often a separate `tmpfs` RAM mount while `/usr/local/bin` lives on `ext4`/`xfs`. `rename(2)` only works within a single mounted filesystem; cross-mount renames fail with `EXDEV`. Always stage the temporary file inside `filepath.Dir(realExePath)` via `os.OpenRoot`. On **Windows**, rename the running `mycli.exe` to `.mycli.exe.old` first, rename the staged file to `mycli.exe`, and clean up `.old` on next launch.

## Complete Post-Quantum (`crypto/mldsa`) Verified Atomic Self-Updater in Go 1.27

```go
package selfupdate

import (
	"crypto/mldsa"
	"crypto/rand"
	"crypto/sha512"
	"errors"
	"fmt"
	"os"
	"path/filepath"
	"runtime"
)

// ApplyVerifiedUpdate verifies the new binary against a NIST FIPS 204 Post-Quantum
// ML-DSA-65 public key (Go 1.27 crypto/mldsa) and atomically replaces the running
// executable on disk using traversal-resistant os.Root.
func ApplyVerifiedUpdate(newBinaryBytes, signature []byte, pubKey *mldsa.PublicKey65) error {
	digest := sha512.Sum512(newBinaryBytes)
	if !pubKey.Verify(digest[:], signature, "mycli-release-v1") {
		return errors.New("selfupdate: ML-DSA-65 cryptographic signature verification failed")
	}

	// Follow symlinks (e.g., Homebrew /opt/homebrew/bin -> ../Cellar/...) before replacing.
	exePath, err := os.Executable()
	if err != nil {
		return fmt.Errorf("locate executable: %w", err)
	}
	realExePath, err := filepath.EvalSymlinks(exePath)
	if err != nil {
		return fmt.Errorf("eval symlinks: %w", err)
	}

	binDir := filepath.Dir(realExePath)
	binName := filepath.Base(realExePath)

	dirRoot, err := os.OpenRoot(binDir)
	if err != nil {
		return fmt.Errorf("open binary root dir %q: %w", binDir, err)
	}
	defer dirRoot.Close()

	tmpName := fmt.Sprintf(".%s.tmp.%s", binName, rand.Text()[:8])
	tmpFile, err := dirRoot.OpenFile(tmpName, os.O_CREATE|os.O_WRONLY|os.O_EXCL, 0755)
	if err != nil {
		return fmt.Errorf("create staging file: %w", err)
	}

	cleanupTmp := true
	defer func() {
		_ = tmpFile.Close()
		if cleanupTmp {
			_ = dirRoot.Remove(tmpName)
		}
	}()

	if _, err := tmpFile.Write(newBinaryBytes); err != nil {
		return fmt.Errorf("write staging binary: %w", err)
	}
	if err := tmpFile.Chmod(0755); err != nil {
		return fmt.Errorf("chmod staging binary: %w", err)
	}
	// Flush dirty pages before rename so power loss never leaves a 0-byte binary.
	if err := tmpFile.Sync(); err != nil {
		return fmt.Errorf("fsync staging binary: %w", err)
	}
	if err := tmpFile.Close(); err != nil {
		return fmt.Errorf("close staging binary: %w", err)
	}

	oldBackupName := "." + binName + ".old"
	if runtime.GOOS == "windows" {
		_ = dirRoot.Remove(oldBackupName)
		if err := dirRoot.Rename(binName, oldBackupName); err != nil {
			return fmt.Errorf("move running windows exe: %w", err)
		}
	}

	if err := dirRoot.Rename(tmpName, binName); err != nil {
		if runtime.GOOS == "windows" {
			_ = dirRoot.Rename(oldBackupName, binName)
		}
		return fmt.Errorf("atomic rename %q -> %q: %w", tmpName, binName, err)
	}

	cleanupTmp = false
	return nil
}
```

## Binary Reachability Auditing with `govulncheck` and `gohawk`

- **Binary call-graph scanning:** Run `govulncheck -mode=binary ./dist/mycli` ([govulncheck](https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck)) to audit compiled release artifacts directly via `.go.buildinfo` and `.gopclntab`, reporting only CVEs on reachable call paths from `main.main`.
- **Resource-leak verification:** Run [gohawk](https://gohawk.dev/) (`go tool gohawk ./...`) to verify that `dirRoot` and `tmpFile` close cleanly on every error branch.

## Related Chapters

- [315 Shipping, Version, Completion, and Release](315-shipping-version-completion.md)
- [999d crypto/mldsa](../07-stdlib/999d-crypto-mldsa.md)
- [175b os.Root Traversal-Resistant Files](../22-security-hardening/175b-os-root-traversal-resistant-files.md)
- [232 Build Cache and Reproducibility](../25-go-deep-dives/232-build-cache-reproducibility.md)
