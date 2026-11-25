---
title: '01. Introduction to Protobuf, gRPC & Protoc'
---

# Introduction to Protobuf, gRPC & Protoc

**Protocol Buffers (Protobuf)** is a strongly typed, schema-first binary serialization format; **gRPC** pairs Protobuf with HTTP/2 multiplexing, deadlines, and code-generated clients and servers across languages.

## Module Syllabus (4 Lessons)

1. What are Protocol Buffers and What are the Benefits?
2. What is gRPC?
3. Types of gRPC APIs
4. `protoc` Tool & Generating Code

## Why Protobuf + gRPC?

- **Compact binary wire format:** Field tags (`1`, `2`, `3`) replace repeated JSON key strings, cutting payload size and CPU serialization cost.
- **Explicit schema contracts:** `.proto` files act as the single source of truth across Go, TypeScript, Python, Rust, and Java services.
- **HTTP/2 transport:** Native header compression (HPACK), multiplexed streams over a single TCP connection, and first-class cancellation/deadlines.

## The Four Types of gRPC APIs

| RPC Type | Signature in `.proto` | Use Case |
|----------|----------------------|----------|
| **Unary** | `rpc GetTodo(GetTodoRequest) returns (GetTodoResponse)` | Standard request/response |
| **Server Streaming** | `rpc WatchTodos(WatchRequest) returns (stream TodoEvent)` | Live feeds, log tails, paginated result streams |
| **Client Streaming** | `rpc UploadFile(stream Chunk) returns (UploadSummary)` | Large file uploads, batched telemetry ingestion |
| **Bi-Directional Streaming** | `rpc Chat(stream ChatMsg) returns (stream ChatMsg)` | Interactive chat, real-time collaborative sessions |

## Generating Go Code with `protoc`

Install the Go compiler plugins (`protoc-gen-go` and `protoc-gen-go-grpc`) and generate stubs from your `.proto` contract:

```bash
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

protoc \
  --go_out=. --go_opt=paths=source_relative \
  --go-grpc_out=. --go-grpc_opt=paths=source_relative \
  proto/todo/v1/todo.proto
```

This produces two files:
- `todo.pb.go` — Go structs and binary marshalling/unmarshalling for messages.
- `todo_grpc.pb.go` — `TodoServiceClient` interface and `TodoServiceServer` interface (plus `UnimplementedTodoServiceServer` for forward compatibility).

## Related Chapters

- Next: [02. Building a Simple gRPC Service](02-simple-grpc-service.md)
- Also see: [09. Using Buf to Manage Protobuf](09-buf-and-connectrpc.md)
