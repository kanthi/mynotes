---
title: '01. PostgreSQL with pgx/v5 and sqlc'
---

# PostgreSQL with `pgx/v5` and `sqlc`

While `database/sql` is great for portable drivers, **`pgx/v5`** unlocks native PostgreSQL features (binary protocol, `COPY`, `pgx.Batch`, `pgxpool`, `LISTEN/NOTIFY`). Pairing `pgx/v5` with **`sqlc`** gives you compile-time-checked SQL queries with zero ORM magic.

## Configuring `pgxpool` for Production

```go
package db

import (
	"context"
	"fmt"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
)

func OpenPool(ctx context.Context, dsn string) (*pgxpool.Pool, error) {
	cfg, err := pgxpool.ParseConfig(dsn)
	if err != nil {
		return nil, fmt.Errorf("parse dsn: %w", err)
	}
	cfg.MaxConns = 25
	cfg.MinConns = 5
	cfg.MaxConnLifetime = 30 * time.Minute
	cfg.MaxConnIdleTime = 5 * time.Minute
	cfg.HealthCheckPeriod = 1 * time.Minute

	pool, err := pgxpool.NewWithConfig(ctx, cfg)
	if err != nil {
		return nil, fmt.Errorf("create pool: %w", err)
	}
	if err := pool.Ping(ctx); err != nil {
		pool.Close()
		return nil, fmt.Errorf("ping db: %w", err)
	}
	return pool, nil
}
```

## Type-Safe Queries with `sqlc`

Write plain SQL in `queries/orders.sql`, run `sqlc generate`, and call generated Go methods that accept `pgx.Tx` or `*pgxpool.Pool` interchangeably via `queries.WithTx(tx)`:

```sql
-- name: CreateOrder :one
INSERT INTO orders (id, customer_email, total_cents)
VALUES ($1, $2, $3)
RETURNING id, customer_email, total_cents, created_at;
```

## Pipelining with `pgx.Batch`

When inserting or updating dozens of rows, `pgx.Batch` sends all queries in a single network round-trip instead of N sequential RTTs—often cutting latency by 5–10×.
