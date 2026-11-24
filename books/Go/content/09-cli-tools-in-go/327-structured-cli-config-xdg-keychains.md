---
title: Structured CLI Configuration, XDG Base Directories, and Secret Keychains
---

# Structured CLI Configuration, XDG Base Directories, and Secret Keychains

Production developer CLIs (`gh`, `kubectl`, `aws`, `docker`, `terraform`) merge layered configuration deterministically, store files in OS-appropriate directories without polluting `$HOME`, write updated settings atomically so concurrent invocations never corrupt files, and keep API tokens out of plain-text dotfiles via OS secret keychains.

## Layered Precedence and XDG Base Directories

```text
Lowest Precedence                                                     Highest Precedence
+------------------+    +------------------+    +------------------+    +--------------+
| Compiled Defaults| -> | User Global Cfg  | -> | Env Variables    | -> | CLI Flags    |
| (DefaultConfig)  |    | ($XDG_CONFIG_HOME|    | (MYCLI_ENDPOINT) |    | (--endpoint) |
+------------------+    |  /mycli/cfg.json)|    +------------------+    +--------------+
                        +------------------+
```

| Directory Type | Go Standard Library API | Linux (XDG Default) | macOS |
|---|---|---|---|
| User Config | `os.UserConfigDir()` | `$XDG_CONFIG_HOME` or `~/.config` | `~/Library/Application Support` |
| User Cache | `os.UserCacheDir()` | `$XDG_CACHE_HOME` or `~/.cache` | `~/Library/Caches` |
| Runtime State / Logs | `$XDG_STATE_HOME` fallback | `~/.local/state` | `~/Library/Application Support` |

> [!TIP]
> On macOS, `os.UserConfigDir()` returns `~/Library/Application Support`. Terminal-first tools (`gh`, `lazygit`, `starship`) check `os.Getenv("XDG_CONFIG_HOME")` first before falling back to `os.UserConfigDir()` so dotfiles remain portable across Linux and macOS.

## Strict Config Parsing with Go 1.27 `encoding/json/v2`

| Parsing Hazard | `encoding/json` (v1) | `encoding/json/v2` (Go 1.27) |
|---|---|---|
| Duplicate JSON keys (`{"role":"admin","role":"user"}`) | Silently overwrites earlier key | **Errors by default** (`jsontext.AllowDuplicateNames(false)`) |
| Case-insensitive typos (`"TimeOut"` vs `"timeout"`) | Silently matches case-insensitively | **Exact case-sensitive match** by default |
| Unknown config keys (`"timout": "5s"`) | Silently ignored unless `DisallowUnknownFields()` | **`json.RejectUnknownMembers(true)`** catches typos immediately |
| Zero structs (`time.Duration`, `time.Time`) | `omitempty` failed to omit zero structs | **`omitzero`** omits zero values cleanly |

## Crash-Safe Atomic Config Writes via `os.OpenRoot`

Calling `os.WriteFile(configPath, data, 0600)` opens the file with `O_TRUNC`, immediately wiping the live config to 0 bytes before writing a single new byte. If `Ctrl+C` or power loss hits mid-write—or another terminal runs `mycli config get` concurrently—the configuration is corrupted.

Instead: open the config directory with `os.OpenRoot(configDir)`, write deterministic JSON into a sibling `0600` temp file, call `tmpFile.Sync()`, close it, and swap atomically with `root.Rename(tmpName, "config.json")`.

## Complete Layered Config Loader and Atomic Saver in Go 1.27

```go
package cliconfig

import (
	"crypto/rand"
	"encoding/json/jsontext"
	"encoding/json/v2"
	"errors"
	"flag"
	"fmt"
	"io/fs"
	"log/slog"
	"net/netip"
	"os"
	"path/filepath"
	"time"
)

type Config struct {
	Endpoint netip.AddrPort `json:"endpoint"`
	Timeout  time.Duration  `json:"timeout"`
	LogLevel slog.Level     `json:"log_level"`
	Profile  string         `json:"profile,omitzero"`
}

func DefaultConfig() Config {
	return Config{
		Endpoint: netip.MustParseAddrPort("127.0.0.1:8443"),
		Timeout:  10 * time.Second,
		LogLevel: slog.LevelInfo,
	}
}

// ResolveConfigDir honors $XDG_CONFIG_HOME first, then falls back to os.UserConfigDir().
func ResolveConfigDir(appName string) (string, error) {
	if xdg := os.Getenv("XDG_CONFIG_HOME"); xdg != "" {
		return filepath.Join(xdg, appName), nil
	}
	base, err := os.UserConfigDir()
	if err != nil {
		return "", fmt.Errorf("resolve user config dir: %w", err)
	}
	return filepath.Join(base, appName), nil
}

// Load merges precedence layers: Defaults < XDG Config (json/v2) < Env Vars < CLI Flags.
// Accepting getenv allows t.Parallel() unit tests without mutating global env vars.
func Load(appName string, args []string, getenv func(string) string) (Config, error) {
	cfg := DefaultConfig()

	cfgDir, err := ResolveConfigDir(appName)
	if err == nil {
		if root, openErr := os.OpenRoot(cfgDir); openErr == nil {
			defer root.Close()
			if raw, readErr := root.ReadFile("config.json"); readErr == nil {
				if jErr := json.Unmarshal(raw, &cfg, json.RejectUnknownMembers(true)); jErr != nil {
					return Config{}, fmt.Errorf("invalid config.json: %w", jErr)
				}
			} else if !errors.Is(readErr, fs.ErrNotExist) {
				return Config{}, fmt.Errorf("read config.json: %w", readErr)
			}
		}
	}

	if envEP := getenv("MYCLI_ENDPOINT"); envEP != "" {
		addr, parseErr := netip.ParseAddrPort(envEP)
		if parseErr != nil {
			return Config{}, fmt.Errorf("invalid MYCLI_ENDPOINT: %w", parseErr)
		}
		cfg.Endpoint = addr
	}
	if envLvl := getenv("MYCLI_LOG_LEVEL"); envLvl != "" {
		if parseErr := cfg.LogLevel.UnmarshalText([]byte(envLvl)); parseErr != nil {
			return Config{}, fmt.Errorf("invalid MYCLI_LOG_LEVEL: %w", parseErr)
		}
	}

	flagSet := flag.NewFlagSet(appName, flag.ContinueOnError)
	flagSet.TextVar(&cfg.Endpoint, "endpoint", cfg.Endpoint, "API endpoint host:port")
	flagSet.DurationVar(&cfg.Timeout, "timeout", cfg.Timeout, "Request timeout")
	flagSet.TextVar(&cfg.LogLevel, "log-level", cfg.LogLevel, "Log level")
	flagSet.StringVar(&cfg.Profile, "profile", cfg.Profile, "Active profile name")

	if err := flagSet.Parse(args); err != nil {
		return Config{}, err
	}

	return cfg, nil
}

// SaveAtomic writes cfg to $XDG_CONFIG_HOME/<app>/config.json with 0600 permissions,
// fsync durability, and atomic rename inside os.Root.
func SaveAtomic(appName string, cfg Config) error {
	cfgDir, err := ResolveConfigDir(appName)
	if err != nil {
		return err
	}
	if err := os.MkdirAll(cfgDir, 0700); err != nil {
		return fmt.Errorf("mkdir config dir: %w", err)
	}

	root, err := os.OpenRoot(cfgDir)
	if err != nil {
		return fmt.Errorf("open config root: %w", err)
	}
	defer root.Close()

	data, err := json.Marshal(cfg, json.Deterministic(true), jsontext.WithIndent("  "))
	if err != nil {
		return fmt.Errorf("marshal config: %w", err)
	}
	data = append(data, '\n')

	tmpName := ".config.json.tmp." + rand.Text()[:8]
	tmpFile, err := root.OpenFile(tmpName, os.O_CREATE|os.O_WRONLY|os.O_EXCL, 0600)
	if err != nil {
		return fmt.Errorf("create temp config: %w", err)
	}

	cleanup := true
	defer func() {
		_ = tmpFile.Close()
		if cleanup {
			_ = root.Remove(tmpName)
		}
	}()

	if _, err := tmpFile.Write(data); err != nil {
		return err
	}
	if err := tmpFile.Sync(); err != nil {
		return err
	}
	if err := tmpFile.Close(); err != nil {
		return err
	}
	if err := root.Rename(tmpName, "config.json"); err != nil {
		return err
	}

	cleanup = false
	return nil
}
```

## OS Secret Keychains and Headless Fallbacks

Never store OAuth refresh tokens or API keys in plain-text dotfiles by default. Using [zalando/go-keyring](https://github.com/zalando/go-keyring):

- **macOS:** Uses `Security.framework` Keychain backed by the login keychain / Secure Enclave.
- **Linux Desktop:** Speaks D-Bus (`org.freedesktop.secrets`) to GNOME Keyring or KWallet.
- **Headless Linux / Containers:** When `DBUS_SESSION_BUS_ADDRESS` is unset, keyring lookups fail—fall back gracefully to an encrypted credential file (Argon2id + `cipher.NewGCMWithRandomNonce`) or a `0600` file with an explicit warning on stderr.

Verify that `root` and `tmpFile` close cleanly on every error path with [gohawk](https://gohawk.dev/) (`go tool gohawk ./...`).

## Related Chapters

- [305 Env Vars and Config Files](305-env-config-files.md)
- [313 Viper and Config Layering](313-viper-config-libs.md)
- [985b encoding/json/v2](../07-stdlib/985b-encoding-json-v2.md)
- [173 Secrets Management and Rotation](../22-security-hardening/173-secrets-management-rotation.md)
