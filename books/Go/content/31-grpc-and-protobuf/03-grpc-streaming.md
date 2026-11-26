---
title: '03. gRPC Streaming'
---

# gRPC Streaming

HTTP/2 lets gRPC send multiple ordered messages over a single RPC call without buffering entire multi-gigabyte payloads in memory.

## Module Syllabus (5 Lessons)

1. Implementing Server Streaming
2. Implementing Client Streaming
3. Implementing Bi-Directional Streaming
4. Exercise — Building a File Upload Application
5. Exercise Solution

## Streaming Patterns at a Glance

![Four gRPC streaming patterns over HTTP/2](../../images/diagram-go-grpc-streaming.svg)

```text
Server Streaming:        Client --- 1 Request ---> Server === N Responses (until nil return) ==> Client
Client Streaming:        Client === N Chunks (Send) ==> Server (Recv until io.EOF) ---> 1 Summary
Bi-Directional Stream:   Client <=== Concurrent Send & Recv over same HTTP/2 stream ===> Server
```

## Client-Streaming File Upload Pattern

For large file uploads, send metadata in the first message (or gRPC metadata headers) and stream fixed-size byte chunks (e.g., 64 KiB) until `io.EOF`, enforcing a maximum byte cap to protect server memory and disk.

```go
package upload

import (
	"crypto/sha256"
	"encoding/hex"
	"io"

	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

const maxUploadBytes = 50 << 20 // 50 MiB

type Chunk struct {
	Filename string
	Data     []byte
}

type UploadSummary struct {
	Filename  string
	SizeBytes int64
	SHA256    string
}

type UploadStream interface {
	Recv() (*Chunk, error)
	SendAndClose(*UploadSummary) error
}

func HandleFileUpload(stream UploadStream) error {
	var (
		filename string
		total    int64
		hasher   = sha256.New()
	)

	for {
		chunk, err := stream.Recv()
		if err == io.EOF {
			return stream.SendAndClose(&UploadSummary{
				Filename:  filename,
				SizeBytes: total,
				SHA256:    hex.EncodeToString(hasher.Sum(nil)),
			})
		}
		if err != nil {
			return status.Errorf(codes.Internal, "receive chunk: %v", err)
		}
		if filename == "" {
			filename = chunk.Filename
		}

		total += int64(len(chunk.Data))
		if total > maxUploadBytes {
			return status.Errorf(codes.ResourceExhausted, "file exceeds %d bytes", maxUploadBytes)
		}
		hasher.Write(chunk.Data)
	}
}
```

## Common Pitfalls

- **Huge chunk sizes:** gRPC defaults to a 4 MiB max message size. Keep stream chunks around 32–64 KiB for smooth backpressure and low memory overhead.
- **Deadlocking bi-directional streams:** Never assume alternating lock-step `Send`/`Recv` unless the protocol guarantees it. Run `Send` and `Recv` in separate goroutines coordinated with `errgroup` and `stream.CloseSend()`.

## Related Chapters

- Previous: [02. Building a Simple gRPC Service](02-simple-grpc-service.md)
- Next: [04. Authentication, SSL/TLS](04-authentication-ssl-tls.md)
