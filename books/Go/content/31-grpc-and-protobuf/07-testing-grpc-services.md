---
title: '07. Testing gRPC Services'
---

# Testing gRPC Services

Test gRPC services at three layers: manual exploratory calls with **Postman** or **`grpcurl`**, fast **unit tests** directly on handler methods, and **in-process end-to-end tests** over `bufconn` that exercise serialization, interceptors, metadata, and streaming without touching host TCP ports.

## Module Syllabus (8 Lessons)

1. Making gRPC Requests via Postman
2. Making gRPC Requests via `grpcurl`
3. Creating Unit Tests for RPCs — Part 1 (Unary)
4. Creating Unit Tests for RPCs — Part 2 (Streaming)
5. Creating Integration/End-to-End Tests (Unary)
6. Creating Integration/End-to-End Tests (Streaming)
7. Exercise — Adding Unit Tests to our TODO Application
8. Exercise Solution

## Exploratory Testing with `grpcurl`

When server reflection (`reflection.Register(srv)`) is enabled in development, `grpcurl` can list services and invoke RPCs directly from the terminal:

```bash
# List services exposed on localhost:50051
grpcurl -plaintext localhost:50051 list

# Invoke a unary RPC with JSON input and metadata header
grpcurl -plaintext \
  -H 'authorization: Bearer dev-token' \
  -d '{"title": "Ship gRPC tests"}' \
  localhost:50051 todo.v1.TodoService/CreateTodo
```

## In-Process E2E Testing with `bufconn`

`google.golang.org/grpc/test/bufconn` creates an in-memory `net.Listener` so your test runs a real `grpc.Server` and `grpc.ClientConn` (including interceptors and streaming) with zero port conflicts and deterministic teardown.

```go
package todo_test

import (
	"context"
	"net"
	"testing"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	"google.golang.org/grpc/test/bufconn"
)

func startBufconnServer(t *testing.T, register func(*grpc.Server)) *grpc.ClientConn {
	t.Helper()
	lis := bufconn.Listen(1 << 20) // 1 MiB in-memory buffer
	srv := grpc.NewServer()
	register(srv)

	go func() {
		_ = srv.Serve(lis)
	}()
	t.Cleanup(func() {
		srv.GracefulStop()
		_ = lis.Close()
	})

	conn, err := grpc.NewClient(
		"passthrough:///bufnet",
		grpc.WithContextDialer(func(ctx context.Context, _ string) (net.Conn, error) {
			return lis.DialContext(ctx)
		}),
		grpc.WithTransportCredentials(insecure.NewCredentials()),
	)
	if err != nil {
		t.Fatalf("dial bufconn: %v", err)
	}
	t.Cleanup(func() { _ = conn.Close() })
	return conn
}
```

## Related Chapters

- Previous: [06. Client Service Config & Load Balancing](06-client-service-config-lb.md)
- Next: [08. Containerising & Deploying to Kubernetes](08-containerising-deploying-k8s.md)
- Book foundations: [Testing Fundamentals](../06-testing/36-testing-fundamentals.md)
