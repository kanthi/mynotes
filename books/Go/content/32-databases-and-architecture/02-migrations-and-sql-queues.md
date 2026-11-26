---
title: '02. Migrations, SQL Job Queues & SQLite WAL'
---

# Migrations, SQL Job Queues & SQLite WAL

Many Go services do not need a separate Redis or RabbitMQ cluster for background jobs—PostgreSQL's `SELECT ... FOR UPDATE SKIP LOCKED` provides a transactional, ACID-consistent job queue, and embedded SQLite in WAL mode handles single-node tools with zero external ops.

## Transactional Job Queue with `SKIP LOCKED`

```sql
WITH next_job AS (
    SELECT id
    FROM jobs
    WHERE status = 'pending' AND run_at <= NOW()
    ORDER BY run_at ASC
    FOR UPDATE SKIP LOCKED
    LIMIT 1
)
UPDATE jobs
SET status = 'running', started_at = NOW(), attempts = attempts + 1
FROM next_job
WHERE jobs.id = next_job.id
RETURNING jobs.id, jobs.payload;
```

Multiple Go worker replicas can execute this query concurrently without blocking each other or claiming the same job twice.

## Schema Migrations (`goose` + `//go:embed`)

Embed SQL migration files directly into your Go binary so deployments are self-contained and never drift from the application code. Always write **expand-and-contract** migrations (add nullable column → backfill → switch code → drop old column) so rolling Kubernetes deployments never fail against the old schema.

## Embedded SQLite in WAL Mode

For CLI tools, edge agents, and single-node services, use `modernc.org/sqlite` (pure Go, `CGO_ENABLED=0`) with WAL mode enabled and a single writer connection (`db.SetMaxOpenConns(1)` for writes) to eliminate `SQLITE_BUSY` errors:

```sql
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA busy_timeout = 5000;
PRAGMA foreign_keys = ON;
```
