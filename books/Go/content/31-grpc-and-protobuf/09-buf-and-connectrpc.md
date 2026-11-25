---
title: '09. Using Buf to Manage Protobuf'
---

# Using Buf to Manage Protobuf

Hand-written `protoc` shell scripts, vendored `.proto` imports, and silent breaking schema changes become painful as teams scale. **Buf** replaces `protoc` plumbing with linting, breaking-change detection, a versioned **Buf Schema Registry (BSR)**, **Protovalidate** rules, and **ConnectRPC** for idiomatic Go and browser clients.

## Module Syllabus (7 Lessons)

1. Challenges with Maintaining Protobuf at Scale & What is Buf?
2. Example Protobuf Registry Walkthrough
3. ConnectRPC — Server & Client Implementation in Go
4. ConnectRPC — Client Implementation for Web (TypeScript / Fetch)
5. Buf Schema Registry, Dependencies & Protovalidate
6. Exercise — Updating our TODO Application to use ConnectRPC
7. Exercise Solution

## Declarative Codegen & Breaking-Change Checks (`buf.yaml` + `buf.gen.yaml`)

```yaml
# buf.yaml
version: v2
modules:
  - path: proto
deps:
  - buf.build/bufbuild/protovalidate
lint:
  use:
    - STANDARD
breaking:
  use:
    - FILE
```

```yaml
# buf.gen.yaml
version: v2
plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen
    opt: paths=source_relative
  - remote: buf.build/connectrpc/go
    out: gen
    opt: paths=source_relative
```

Run `buf lint`, `buf breaking --against '.git#branch=main'`, and `buf generate` in CI so breaking `.proto` changes are caught before merge.

## ConnectRPC Server on Standard `net/http`

ConnectRPC speaks **gRPC**, **gRPC-Web**, and **Connect (JSON/Protobuf over HTTP/1.1 or HTTP/2)** simultaneously on a standard Go `http.ServeMux`—eliminating the need for an Envoy gRPC-Web proxy for browser clients.

```go
package main

import (
	"context"
	"net/http"

	"connectrpc.com/connect"
	"golang.org/x/net/http2"
	"golang.org/x/net/http2/h2c"
)

// Connect handlers use typed connect.Request[T] and connect.Response[T] over standard net/http.
type TodoConnectServer struct{}

func serveConnect(mux *http.ServeMux) error {
	// h2c enables HTTP/2 without TLS for local dev or behind a TLS-terminating Ingress.
	return http.ListenAndServe(":8080", h2c.NewHandler(mux, &http2.Server{}))
}

// Protovalidate enforces schema constraints declared directly in .proto files,
// such as: string title = 1 [(buf.validate.field).string.min_len = 1];
var _ = context.Background
var _ = connect.CodeInvalidArgument
```

## Related Chapters

- Previous: [08. Containerising & Deploying to Kubernetes](08-containerising-deploying-k8s.md)
- Section Overview: [gRPC and Protobuf Intro](index.md)
- Also see: [gRPC Microservices Overview](../16-web/62-grpc-microservices.md)
