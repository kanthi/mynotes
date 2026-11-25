---
title: '102b. Project: High-Throughput OTel Ingestor & ClickHouse Go Benchmark Suite'
---

# Project: High-Throughput OTel Ingestor & ClickHouse Go Benchmark Suite

Build a high-throughput Go telemetry ingestion service that accepts structured logs/traces over gRPC/Protobuf, applies bounded worker pools and backpressure, batches columnar writes into **ClickHouse** via `ch-go` with mTLS, and enforces a **CI benchmark gate** that fails PRs on >5% throughput or allocation regressions.

## Architecture

```text
OTel SDKs / Agents (gRPC + Protobuf)
        |
        v
[ Go Ingestor ]
   ├── Bounded Channel Ring (backpressure -> returns codes.ResourceExhausted when full)
   ├── Batcher Workers (flush every 5,000 rows OR 250ms, whichever comes first)
   └── ch-go Columnar Writer (mTLS, ZSTD compression, OTel self-tracing)
        |
        v
   ClickHouse (MergeTree partitioned by toYYYYMM(timestamp))
```

## Bounded Batcher with Flush-on-Size-or-Timer

```go
package ingest

import (
	"context"
	"errors"
	"sync"
	"time"
)

var ErrOverloaded = errors.New("ingestor queue full")

type LogRecord struct {
	Timestamp time.Time
	Service   string
	Severity  string
	Body      string
}

type FlushFunc func(ctx context.Context, batch []LogRecord) error

type Batcher struct {
	ch        chan LogRecord
	maxBatch  int
	flushWait time.Duration
	flush     FlushFunc
	wg        sync.WaitGroup
}

func NewBatcher(queueCap, maxBatch int, flushWait time.Duration, flush FlushFunc) *Batcher {
	return &Batcher{
		ch:        make(chan LogRecord, queueCap),
		maxBatch:  maxBatch,
		flushWait: flushWait,
		flush:     flush,
	}
}

func (b *Batcher) Enqueue(rec LogRecord) error {
	select {
	case b.ch <- rec:
		return nil
	default:
		return ErrOverloaded
	}
}

func (b *Batcher) Start(ctx context.Context, workers int) {
	for range workers {
		b.wg.Add(1)
		go func() {
			defer b.wg.Done()
			b.worker(ctx)
		}()
	}
}

func (b *Batcher) worker(ctx context.Context) {
	buf := make([]LogRecord, 0, b.maxBatch)
	timer := time.NewTimer(b.flushWait)
	defer timer.Stop()

	doFlush := func() {
		if len(buf) == 0 {
			return
		}
		_ = b.flush(ctx, buf)
		buf = buf[:0] // reuse backing array to keep allocs near zero
	}

	for {
		select {
		case <-ctx.Done():
			doFlush()
			return
		case rec := <-b.ch:
			buf = append(buf, rec)
			if len(buf) >= b.maxBatch {
				doFlush()
				timer.Reset(b.flushWait)
			}
		case <-timer.C:
			doFlush()
			timer.Reset(b.flushWait)
		}
	}
}
```

## CI Benchmark Regression Gate

```bash
go test -bench=BenchmarkBatchInsert -benchmem -count=8 ./ingest > pr.txt
benchstat main.txt pr.txt
# Fail CI if ns/op or allocs/op regresses by > 5%
```

Profile hot paths with `go test -cpuprofile=cpu.out -memprofile=mem.out -trace=trace.out` to eliminate per-row heap escapes.
