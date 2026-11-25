---
title: 'Project: HTTP Debug CLI (httpdbg)'
---

# Project: HTTP Debug CLI (`httpdbg`)

Build a zero-dependency `curl`-inspired HTTP diagnostic CLI in Go that instruments DNS lookup, TCP connect, TLS handshake, time-to-first-byte (TTFB), and total latency using `net/http/httptrace`, with optional JSON output for scripting.

## Architecture

```text
httpdbg -method GET -header "Accept: application/json" -json https://example.com
   ├── Parse flags & headers into FlagSet
   ├── Attach httptrace.ClientTrace to request context
   ├── Execute request with explicit timeout & custom Transport (DisableKeepAlives)
   └── Print phase timings + status + response headers (text or JSON)
```

## Complete Runnable Implementation

```go
package main

import (
	"context"
	"crypto/tls"
	"encoding/json"
	"flag"
	"fmt"
	"io"
	"net/http"
	"net/http/httptrace"
	"os"
	"strings"
	"time"
)

type headerFlags []string

func (h *headerFlags) String() string { return strings.Join(*h, ", ") }
func (h *headerFlags) Set(v string) error {
	*h = append(*h, v)
	return nil
}

type Report struct {
	URL        string        `json:"url"`
	Method     string        `json:"method"`
	StatusCode int           `json:"status_code"`
	Proto      string        `json:"proto"`
	DNS        time.Duration `json:"dns_ns"`
	Connect    time.Duration `json:"connect_ns"`
	TLS        time.Duration `json:"tls_ns"`
	TTFB       time.Duration `json:"ttfb_ns"`
	Total      time.Duration `json:"total_ns"`
	BodyBytes  int64         `json:"body_bytes"`
}

func probe(ctx context.Context, method, rawURL string, headers []string, timeout time.Duration) (Report, error) {
	ctx, cancel := context.WithTimeout(ctx, timeout)
	defer cancel()

	var dnsStart, connStart, tlsStart, reqStart time.Time
	var rep Report
	rep.URL = rawURL
	rep.Method = method

	trace := &httptrace.ClientTrace{
		DNSStart:             func(httptrace.DNSStartInfo) { dnsStart = time.Now() },
		DNSDone:              func(httptrace.DNSDoneInfo) { rep.DNS = time.Since(dnsStart) },
		ConnectStart:         func(_, _ string) { connStart = time.Now() },
		ConnectDone:          func(_, _ string, _ error) { rep.Connect = time.Since(connStart) },
		TLSHandshakeStart:    func() { tlsStart = time.Now() },
		TLSHandshakeDone:     func(tls.ConnectionState, error) { rep.TLS = time.Since(tlsStart) },
		GotFirstResponseByte: func() { rep.TTFB = time.Since(reqStart) },
	}

	req, err := http.NewRequestWithContext(httptrace.WithClientTrace(ctx, trace), method, rawURL, nil)
	if err != nil {
		return rep, err
	}
	for _, h := range headers {
		k, v, ok := strings.Cut(h, ":")
		if !ok {
			return rep, fmt.Errorf("invalid header %q (expected Key: Value)", h)
		}
		req.Header.Add(strings.TrimSpace(k), strings.TrimSpace(v))
	}

	client := &http.Client{
		Transport: &http.Transport{DisableKeepAlives: true},
	}

	reqStart = time.Now()
	resp, err := client.Do(req)
	if err != nil {
		return rep, err
	}
	defer resp.Body.Close()

	n, err := io.Copy(io.Discard, resp.Body)
	if err != nil {
		return rep, err
	}
	rep.Total = time.Since(reqStart)
	rep.StatusCode = resp.StatusCode
	rep.Proto = resp.Proto
	rep.BodyBytes = n
	return rep, nil
}

func main() {
	var headers headerFlags
	method := flag.String("method", http.MethodGet, "HTTP method")
	timeout := flag.Duration("timeout", 10*time.Second, "request timeout")
	asJSON := flag.Bool("json", false, "output JSON report")
	flag.Var(&headers, "header", "HTTP header 'Key: Value' (repeatable)")
	flag.Parse()

	if flag.NArg() != 1 {
		fmt.Fprintln(os.Stderr, "usage: httpdbg [flags] <url>")
		os.Exit(2)
	}

	rep, err := probe(context.Background(), *method, flag.Arg(0), headers, *timeout)
	if err != nil {
		fmt.Fprintf(os.Stderr, "error: %v\n", err)
		os.Exit(1)
	}

	if *asJSON {
		enc := json.NewEncoder(os.Stdout)
		enc.SetIndent("", "  ")
		_ = enc.Encode(rep)
		return
	}

	fmt.Printf("%s %s -> %d (%s, %d bytes)\n", rep.Method, rep.URL, rep.StatusCode, rep.Proto, rep.BodyBytes)
	fmt.Printf("  DNS:     %s\n  Connect: %s\n  TLS:     %s\n  TTFB:    %s\n  Total:   %s\n",
		rep.DNS.Round(time.Millisecond),
		rep.Connect.Round(time.Millisecond),
		rep.TLS.Round(time.Millisecond),
		rep.TTFB.Round(time.Millisecond),
		rep.Total.Round(time.Millisecond),
	)
}
```

## Verification & Extensions

- Run against a local `httptest.NewTLSServer` in `go test -race ./...` to verify all timing phases populate without external internet access.
- Extend with `-head` (print response headers), `-insecure` (skip TLS verification for debugging expired certs), and `-count N` to compute p50/p95/p99 latency distributions.
