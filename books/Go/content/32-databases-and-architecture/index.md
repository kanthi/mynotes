---
title: Intro
---

# Intro

This part fills four high-value production domains in modern Go backend engineering: **database engineering** (`pgx/v5`, `sqlc`, migrations, SQL job queues, SQLite WAL), **pragmatic Clean/Hexagonal architecture & DDD**, **durable workflow execution (Temporal)**, and **modern real-time/edge protocols** (SSE, HTTP/3 QUIC, WebRTC).

## Chapter Map

| # | Topic | Focus | Chapter |
|---|-------|-------|---------|
| 01 | **PostgreSQL with `pgx/v5` & `sqlc`** | Connection pools, batching, type-safe generated SQL | [01. PostgreSQL with `pgx/v5` & `sqlc`](01-pgx-and-sqlc.md) |
| 02 | **Migrations, SQL Job Queues & SQLite WAL** | `goose`, isolation levels, `SKIP LOCKED`, embedded SQLite | [02. Migrations, SQL Job Queues & SQLite WAL](02-migrations-and-sql-queues.md) |
| 03 | **Clean Architecture & Pragmatic DDD** | Hexagonal ports/adapters, package-by-feature, aggregates | [03. Clean Architecture & Pragmatic DDD](03-clean-architecture-and-ddd.md) |
| 04 | **Durable Workflows with Temporal** | Workflows, activities, deterministic replay, signals | [04. Durable Workflows with Temporal](04-durable-workflows-temporal.md) |
| 05 | **Real-Time & Edge Protocols** | Server-Sent Events (SSE), HTTP/3 QUIC, WebRTC (`pion`) | [05. Real-Time & Edge Protocols](05-realtime-sse-quic-webrtc.md) |
