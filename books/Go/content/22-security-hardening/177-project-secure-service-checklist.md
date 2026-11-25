---
title: 'Project: Secure Service Checklist & Hardened Skeleton'
---

# Project: Secure Service Checklist & Hardened Skeleton

Turn security hardening principles into a concrete, repeatable Go HTTP service skeleton with **strict timeouts**, **body size limits**, **security headers**, **CSRF protection**, **TLS 1.3**, and **automated CI checks**.

## Hardened HTTP Server Skeleton

```go
package main

import (
	"crypto/tls"
	"net/http"
	"time"
)

func securityHeaders(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("X-Content-Type-Options", "nosniff")
		w.Header().Set("X-Frame-Options", "DENY")
		w.Header().Set("Referrer-Policy", "strict-origin-when-cross-origin")
		w.Header().Set("Content-Security-Policy", "default-src 'self'")
		// Cap request body to 1 MiB to prevent memory exhaustion DoS.
		r.Body = http.MaxBytesReader(w, r.Body, 1<<20)
		next.ServeHTTP(w, r)
	})
}

func newHardenedServer(addr string, handler http.Handler) *http.Server {
	csrf := http.NewCrossOriginProtection()
	return &http.Server{
		Addr:              addr,
		Handler:           securityHeaders(csrf.Handler(handler)),
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       15 * time.Second,
		WriteTimeout:      15 * time.Second,
		IdleTimeout:       60 * time.Second,
		MaxHeaderBytes:    1 << 16, // 64 KiB
		TLSConfig: &tls.Config{
			MinVersion: tls.VersionTLS13,
		},
	}
}
```

## Production Security Checklist

| Layer | Requirement | Verification Command / Check |
|-------|-------------|------------------------------|
| **Transport** | TLS 1.3 minimum, HSTS on HTTPS origins | `sslscan` / `tls.VersionTLS13` unit test |
| **Input & Edge** | `http.MaxBytesReader`, `http.NewCrossOriginProtection`, `os.Root` for file access | Oversized body returns `413`; cross-site POST rejected |
| **Auth & Secrets** | Short-lived JWTs with issuer/audience checks; secrets from env/vault, never logged | `slog` redaction test on `Token` types |
| **Dependencies** | Zero known reachable CVEs; pinned `go.sum` | `govulncheck ./...` in CI |
| **Static Audit** | No SQL string concatenation, no weak `math/rand` for tokens | `gosec ./...` and `go vet ./...` |
| **Container** | Non-root user (`USER 65532:65532`), read-only rootfs, `CGO_ENABLED=0` | `docker inspect` + Kubernetes `securityContext` |
