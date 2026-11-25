---
title: '102g. Project: Cross-Platform Telemetry Agent & HIL Fleet Health Exporter'
---

# Project: Cross-Platform Telemetry Agent & HIL Fleet Health Exporter

Build a lightweight Go daemon (`systemd` unit on Linux, `launchd` on macOS) that collects host, network, and hardware-in-the-loop (HIL) health metrics—interface stats, DNS latency, socket states, SMART/NVMe wear, NIC link drops, and `systemd` unit failures—buffers locally in **SQLite WAL** during network outages, and streams over **mTLS + gRPC** with exponential backoff.

## Architecture

```text
Host Collectors (15s tick)
   ├── /proc/net/dev (RX/TX drops, errors)
   ├── DNS probe + TCP socket states
   ├── SMART/NVMe wear & systemd failed units
   └── Prometheus /metrics endpoint (local scrape)
        |
        v
[ SQLite WAL Ring Buffer (disk-backed during WAN outage) ]
        |
        v  mTLS + gRPC Stream (exponential backoff + jitter, graceful ctx shutdown)
[ Central Telemetry Collector ]
```

## Offline-Resilient SQLite WAL Buffer Pattern

```go
package agent

import (
	"context"
	"database/sql"
	"fmt"
	"time"
)

type Sample struct {
	ID        int64
	CreatedAt time.Time
	Payload   []byte
}

func InitBuffer(ctx context.Context, db *sql.DB) error {
	const ddl = `
		PRAGMA journal_mode = WAL;
		PRAGMA synchronous = NORMAL;
		CREATE TABLE IF NOT EXISTS pending_samples (
			id INTEGER PRIMARY KEY AUTOINCREMENT,
			created_at INTEGER NOT NULL,
			payload BLOB NOT NULL
		);
	`
	_, err := db.ExecContext(ctx, ddl)
	return err
}

func EnqueueSample(ctx context.Context, db *sql.DB, payload []byte) error {
	_, err := db.ExecContext(ctx,
		`INSERT INTO pending_samples (created_at, payload) VALUES (?, ?)`,
		time.Now().Unix(), payload,
	)
	return err
}

func DrainBatch(ctx context.Context, db *sql.DB, limit int, send func(context.Context, []Sample) error) error {
	rows, err := db.QueryContext(ctx,
		`SELECT id, created_at, payload FROM pending_samples ORDER BY id ASC LIMIT ?`, limit,
	)
	if err != nil {
		return fmt.Errorf("select pending: %w", err)
	}
	defer rows.Close()

	var batch []Sample
	var maxID int64
	for rows.Next() {
		var s Sample
		var ts int64
		if err := rows.Scan(&s.ID, &ts, &s.Payload); err != nil {
			return err
		}
		s.CreatedAt = time.Unix(ts, 0)
		maxID = s.ID
		batch = append(batch, s)
	}
	if len(batch) == 0 || rows.Err() != nil {
		return rows.Err()
	}

	if err := send(ctx, batch); err != nil {
		return err // keep rows in SQLite for retry
	}
	_, err = db.ExecContext(ctx, `DELETE FROM pending_samples WHERE id <= ?`, maxID)
	return err
}
```

## Acceptance Drills

- Block outbound port `50051` with `nftables` for 2 minutes, verify samples accumulate in SQLite without memory growth, restore connectivity, and verify all buffered samples drain in order.
- Send `SIGTERM` to the `systemd` unit and confirm clean flush + exit within 5 seconds.
