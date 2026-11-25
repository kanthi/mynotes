---
title: 'Project: Instrumented Mini-Service'
---

# Project: Instrumented Mini-Service

Wire the three observability pillars—**structured `slog` logs with request IDs**, **Prometheus counters/histograms**, and **health/readiness probes**—into a runnable Go HTTP service.

## Architecture

```text
Incoming HTTP Request
   ├── RequestID Middleware (extract/generate X-Request-ID -> ctx + response header)
   ├── Metrics Middleware (status counter + duration histogram)
   ├── Structured Logging (slog JSON line with request_id, method, path, status, ms)
   └── Endpoints: /api/work, /healthz, /readyz, /metrics
```

## Complete Runnable Service

```go
package main

import (
	"context"
	"log/slog"
	"net/http"
	"os"
	"sync/atomic"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

type statusRecorder struct {
	http.ResponseWriter
	status int
}

func (r *statusRecorder) WriteHeader(code int) {
	r.status = code
	r.ResponseWriter.WriteHeader(code)
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo}))
	var ready atomic.Bool
	ready.Store(true)

	reqTotal := prometheus.NewCounterVec(prometheus.CounterOpts{
		Name: "http_requests_total",
		Help: "Total HTTP requests by route and status.",
	}, []string{"path", "status"})
	reqDuration := prometheus.NewHistogramVec(prometheus.HistogramOpts{
		Name:    "http_request_duration_seconds",
		Help:    "HTTP request latency in seconds.",
		Buckets: prometheus.DefBuckets,
	}, []string{"path"})
	prometheus.MustRegister(reqTotal, reqDuration)

	instrument := func(path string, next http.HandlerFunc) http.HandlerFunc {
		return func(w http.ResponseWriter, r *http.Request) {
			start := time.Now()
			rec := &statusRecorder{ResponseWriter: w, status: http.StatusOK}
			next(rec, r)
			dur := time.Since(start)
			reqTotal.WithLabelValues(path, http.StatusText(rec.status)).Inc()
			reqDuration.WithLabelValues(path).Observe(dur.Seconds())
			logger.Info("http request",
				"method", r.Method,
				"path", path,
				"status", rec.status,
				"duration_ms", dur.Milliseconds(),
			)
		}
	}

	mux := http.NewServeMux()
	mux.HandleFunc("GET /healthz", func(w http.ResponseWriter, _ *http.Request) { w.WriteHeader(http.StatusOK) })
	mux.HandleFunc("GET /readyz", func(w http.ResponseWriter, _ *http.Request) {
		if !ready.Load() {
			http.Error(w, "not ready", http.StatusServiceUnavailable)
			return
		}
		w.WriteHeader(http.StatusOK)
	})
	mux.Handle("GET /metrics", promhttp.Handler())
	mux.HandleFunc("GET /api/work", instrument("/api/work", func(w http.ResponseWriter, _ *http.Request) {
		time.Sleep(15 * time.Millisecond)
		_, _ = w.Write([]byte(`{"ok":true}`))
	}))

	srv := &http.Server{Addr: ":8080", Handler: mux, ReadHeaderTimeout: 5 * time.Second}
	logger.Info("listening", "addr", srv.Addr)
	_ = srv.ListenAndServe()
	_ = srv.Shutdown(context.Background())
}
```

## Verification

- Hit `GET /api/work` and confirm a structured JSON log line appears on `stdout`.
- Scrape `GET /metrics` and verify `http_requests_total` and `http_request_duration_seconds_bucket` increment.
- Flip `ready.Store(false)` and verify `GET /readyz` returns `503 Service Unavailable` while `GET /healthz` stays `200 OK`.
