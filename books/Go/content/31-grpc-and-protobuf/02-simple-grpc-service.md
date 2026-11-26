---
title: '02. Building a Simple gRPC Service'
---

# Building a Simple gRPC Service

Build a complete unary gRPC service end to end—defining the `.proto` contract, implementing server and client, returning rich gRPC status errors, and running the server cleanly with graceful shutdown.

## Module Syllabus (7 Lessons)

1. Defining a gRPC Service Contract
2. Implementing a Server
3. Implementing a Client
4. Error Handling (`google.golang.org/grpc/status` & `codes`)
5. Exercise — Building a TODO application
6. Exercise Solution
7. Running a Server Correctly (graceful shutdown, reflection, health service)

## Defining the Contract (`todo.proto`)

```protobuf
syntax = "proto3";

package todo.v1;

option go_package = "example.com/todogrpc/gen/todo/v1;todov1";

service TodoService {
  rpc CreateTodo(CreateTodoRequest) returns (CreateTodoResponse);
  rpc GetTodo(GetTodoRequest) returns (GetTodoResponse);
}

message Todo {
  string id = 1;
  string title = 2;
  bool done = 3;
}

message CreateTodoRequest {
  string title = 1;
}

message CreateTodoResponse {
  Todo todo = 1;
}

message GetTodoRequest {
  string id = 1;
}

message GetTodoResponse {
  Todo todo = 1;
}
```

## Server Implementation, Error Handling & Graceful Shutdown

Always embed `UnimplementedTodoServiceServer` for forward compatibility and translate domain errors into canonical gRPC `codes.*` values (`codes.InvalidArgument`, `codes.NotFound`, `codes.Internal`).

```go
package main

import (
	"context"
	"log/slog"
	"net"
	"os"
	"os/signal"
	"sync"
	"syscall"

	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/health"
	healthpb "google.golang.org/grpc/health/grpc_health_v1"
	"google.golang.org/grpc/reflection"
	"google.golang.org/grpc/status"
)

type TodoServer struct {
	// Embed generated UnimplementedTodoServiceServer here.
	mu    sync.RWMutex
	items map[string]string
}

func (s *TodoServer) validateTitle(title string) error {
	if title == "" {
		return status.Error(codes.InvalidArgument, "title must not be empty")
	}
	return nil
}

func runServer(ctx context.Context, addr string) error {
	lis, err := net.Listen("tcp", addr)
	if err != nil {
		return err
	}

	srv := grpc.NewServer()
	// Register standard gRPC health check and reflection (for grpcurl/Postman).
	healthSrv := health.NewServer()
	healthpb.RegisterHealthServer(srv, healthSrv)
	reflection.Register(srv)

	go func() {
		<-ctx.Done()
		slog.Info("gracefully stopping gRPC server")
		srv.GracefulStop()
	}()

	slog.Info("gRPC server listening", "addr", addr)
	return srv.Serve(lis)
}

func main() {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()
	if err := runServer(ctx, ":50051"); err != nil {
		slog.Error("server exited with error", "err", err)
		os.Exit(1)
	}
}
```

## Calling the Service from a Go Client

Always reuse a single `*grpc.ClientConn` across goroutines—it manages HTTP/2 connection pooling and reconnection internally.

```go
package client

import (
	"context"
	"fmt"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/credentials/insecure"
	"google.golang.org/grpc/status"
)

func DialAndInspectError(addr string) error {
	conn, err := grpc.NewClient(addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		return fmt.Errorf("new client: %w", err)
	}
	defer conn.Close()

	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancel()
	_ = ctx

	// Inspect structured gRPC status codes on the client side:
	if err != nil {
		st, ok := status.FromError(err)
		if ok && st.Code() == codes.InvalidArgument {
			return fmt.Errorf("validation error: %s", st.Message())
		}
		return err
	}
	return nil
}
```

## Related Chapters

- Previous: [01. Introduction to Protobuf, gRPC & Protoc](01-protobuf-grpc-protoc.md)
- Next: [03. gRPC Streaming](03-grpc-streaming.md)
