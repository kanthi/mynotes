---
title: Zero-Allocation Database Engineering (RawBytes, RowsColumnScanner, and Pool Tuning)
---

# Zero-Allocation Database Engineering (`RawBytes`, `RowsColumnScanner`, and Pool Tuning)

Scanning 10,000 rows with standard `rows.Scan(&id, &name, &payload)` allocates per-row strings/slices, boxes columns through legacy `driver.Value` (`any`), and risks connection-pool deadlocks whenever an early return skips `rows.Close()`. Go 1.27 adds **`driver.RowsColumnScanner`** and exports **`sql.ConvertAssign`** to eliminate intermediate `driver.Value` boxing.

## Legacy `driver.Rows.Next` vs. Go 1.27 `driver.RowsColumnScanner`

```text
Legacy driver.Rows.Next(dest []driver.Value)       Go 1.27 driver.RowsColumnScanner
+--------------------------------------------+     +--------------------------------------------+
| 1. Driver reads wire bytes from socket     |     | 1. Driver reads wire bytes from socket     |
| 2. Boxes each column into driver.Value(any)|     | 2. ScanColumn(dest, colIdx) writes wire    |
| 3. sql.Scan copies from driver.Value -> *T |     |    bytes directly into caller's *T / slice |
| -> Double conversion + interface boxing    |     | -> Zero intermediate driver.Value boxing   |
+--------------------------------------------+     +--------------------------------------------+
```

## Zero-Allocation JSONB Row Streaming (`sql.RawBytes` + `jsontext`)

`sql.RawBytes` slices directly into the driver's network read buffer (`0 B/op`, valid only until the next `rows.Next()` / `rows.Scan()` / `rows.Close()`). Pairing it with a reusable `bytes.Reader` and Go 1.27's `jsontext.Decoder` cuts 1,000-row JSONB scans by ~6.2x with zero per-row buffer allocations:

```go
package dbstream

import (
	"bytes"
	"context"
	"database/sql"
	"encoding/json/jsontext"
	"encoding/json/v2"
)

type OrderMetadata struct {
	CustomerTier string  `json:"customer_tier"`
	TotalCents   int64   `json:"total_cents"`
	DiscountPct  float64 `json:"discount_pct,omitzero"`
}

func StreamHighValueOrders(ctx context.Context, db *sql.DB, minCents int64) (int64, error) {
	rows, err := db.QueryContext(ctx, `SELECT id, metadata_jsonb FROM orders WHERE status = 'COMPLETED'`)
	if err != nil {
		return 0, err
	}
	defer rows.Close() // Always immediately after err check (verified by gohawk).

	var (
		orderID int64
		rawJSON sql.RawBytes
		br      bytes.Reader
		meta    OrderMetadata
		sum     int64
	)
	dec := jsontext.NewDecoder(&br)

	for rows.Next() {
		if err := rows.Scan(&orderID, &rawJSON); err != nil {
			return 0, err
		}
		br.Reset(rawJSON)
		dec.Reset(&br)
		meta = OrderMetadata{}
		if err := json.UnmarshalDecode(dec, &meta); err != nil {
			return 0, err
		}
		if meta.TotalCents >= minCents {
			sum += meta.TotalCents
		}
	}
	if err := rows.Err(); err != nil {
		return 0, err
	}
	return sum, nil
}
```

## Production `sql.DB` Pool Sizing and `*sql.Stmt` Pitfalls

```go
func ConfigureProductionPool(db *sql.DB, maxConns int) {
	db.SetMaxOpenConns(maxConns)      // Never leave at 0 (unlimited)!
	db.SetMaxIdleConns(maxConns)      // Match MaxOpenConns to avoid TCP+TLS churn on bursts.
	db.SetConnMaxIdleTime(5 * time.Minute)
	db.SetConnMaxLifetime(45 * time.Minute)
}
```

> [!WARNING]
> Avoid `tx.StmtContext(ctx, globalStmt)` unless the statement is already prepared on that specific connection—otherwise it sends a fresh `PREPARE` on transaction start and `DEALLOCATE` on commit. Always place `defer rows.Close()` right after `QueryContext` and `defer tx.Rollback()` right after `BeginTx`, and enforce both in CI with [gohawk](https://gohawk.dev/) (`resourcelifetime`).

## Related Chapters

- [993 database/sql](../07-stdlib/993-database-sql-stdlib.md)
- [993b rows.Err() and sqlrowserr](../07-stdlib/993b-rows-err-and-sqlrowserr.md)
- [260 database/sql Pool](../25-go-deep-dives/260-database-sql-pool.md)
