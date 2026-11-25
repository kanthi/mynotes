---
title: 'Project: netcheck Network Diagnostic Toolkit'
---

# Project: `netcheck` Network Diagnostic Toolkit

Build a concurrent multi-target network diagnostic CLI that runs **DNS resolution**, **TCP dial**, **TLS certificate expiry**, and **HTTP status** checks against a list of targets with bounded concurrency and per-target timeout budgets.

## Architecture

```text
targets (host:port / URL)
   |
   v  errgroup (SetLimit = concurrency)
+--------------------------------------------------+
| 1. DNS LookupHost (net.DefaultResolver)          |
| 2. TCP DialContext (net.Dialer)                  |
| 3. TLS Handshake -> cert expiry & subject        |
| 4. HTTP HEAD/GET -> status code & TTFB           |
+--------------------------------------------------+
   |
   v
Sorted summary table / JSON + non-zero exit code if any target failed
```

## Core Implementation

```go
package main

import (
	"context"
	"crypto/tls"
	"encoding/json"
	"fmt"
	"net"
	"os"
	"time"

	"golang.org/x/sync/errgroup"
)

type CheckResult struct {
	Target     string        `json:"target"`
	IPs        []string      `json:"ips,omitempty"`
	TCPReach   bool          `json:"tcp_reachable"`
	TLSExpiry  *time.Time    `json:"tls_expiry,omitempty"`
	Latency    time.Duration `json:"latency_ms"`
	Err        string        `json:"error,omitempty"`
}

func checkTarget(ctx context.Context, hostPort string, timeout time.Duration) CheckResult {
	ctx, cancel := context.WithTimeout(ctx, timeout)
	defer cancel()

	start := time.Now()
	res := CheckResult{Target: hostPort}

	host, _, err := net.SplitHostPort(hostPort)
	if err != nil {
		res.Err = err.Error()
		return res
	}

	ips, err := net.DefaultResolver.LookupHost(ctx, host)
	if err != nil {
		res.Err = fmt.Sprintf("dns: %v", err)
		return res
	}
	res.IPs = ips

	dialer := &net.Dialer{Timeout: timeout}
	conn, err := dialer.DialContext(ctx, "tcp", hostPort)
	if err != nil {
		res.Err = fmt.Sprintf("tcp: %v", err)
		return res
	}
	res.TCPReach = true

	tlsConn := tls.Client(conn, &tls.Config{ServerName: host, MinVersion: tls.VersionTLS12})
	if err := tlsConn.HandshakeContext(ctx); err == nil {
		certs := tlsConn.ConnectionState().PeerCertificates
		if len(certs) > 0 {
			exp := certs[0].NotAfter
			res.TLSExpiry = &exp
		}
	}
	_ = tlsConn.Close()

	res.Latency = time.Since(start)
	return res
}

func main() {
	targets := os.Args[1:]
	if len(targets) == 0 {
		fmt.Fprintln(os.Stderr, "usage: netcheck host:port [host:port ...]")
		os.Exit(2)
	}

	results := make([]CheckResult, len(targets))
	g, ctx := errgroup.WithContext(context.Background())
	g.SetLimit(8)

	for i, t := range targets {
		g.Go(func() error {
			results[i] = checkTarget(ctx, t, 5*time.Second)
			return nil
		})
	}
	_ = g.Wait()

	enc := json.NewEncoder(os.Stdout)
	enc.SetIndent("", "  ")
	_ = enc.Encode(results)

	for _, r := range results {
		if r.Err != "" {
			os.Exit(1)
		}
	}
}
```

## Verification Checklist

- Test against a closed local port (`127.0.0.1:1`) to confirm fast TCP failure and non-zero exit code.
- Test against a local `httptest.NewTLSServer` to verify certificate expiry extraction.
- Run with `go test -race ./...` to confirm zero data races on `results[i]`.
