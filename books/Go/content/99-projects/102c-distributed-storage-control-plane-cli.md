---
title: '102c. Project: Distributed Object Storage Control Plane & Multi-Arch CLI'
---

# Project: Distributed Object Storage Control Plane & Multi-Arch CLI

Build a **gRPC + REST Go control plane** and **CLI** that splits objects into fixed-size content-addressed chunks, replicates them across worker storage nodes with quorum acknowledgments, runs background health/scrubbing checks, and ships as multi-arch (`linux/amd64`, `linux/arm64`) containers on Kubernetes with Prometheus metrics.

## Architecture

```text
objctl CLI (put / get / stat / health)
        |  gRPC (streaming chunks) or HTTP/REST
        v
[ Control Plane (Go) ] --- Metadata Store (PostgreSQL / etcd)
   ├── Consistent Hash Ring -> selects N=3 storage nodes per chunk
   ├── Quorum Write (W=2) & Streaming Read with SHA-256 verification
   └── Background Health Loop -> marks dead nodes & triggers re-replication
        |
   +----+----+----+
   v         v    v
[Node 1] [Node 2] [Node 3]  (Worker Storage Nodes exposing gRPC + /metrics)
```

## Content-Addressed Chunking & Quorum Replication

```go
package storage

import (
	"context"
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"io"
	"sync/atomic"

	"golang.org/x/sync/errgroup"
)

type NodeClient interface {
	PutChunk(ctx context.Context, digest string, data []byte) error
}

type ChunkMeta struct {
	Index  int
	Digest string
	Size   int
	Nodes  []string
}

func ReplicateStream(
	ctx context.Context,
	r io.Reader,
	chunkSize int,
	pickNodes func(digest string) ([]NodeClient, []string),
	quorum int,
) ([]ChunkMeta, error) {
	buf := make([]byte, chunkSize)
	var manifest []ChunkMeta

	for idx := 0; ; idx++ {
		n, err := io.ReadFull(r, buf)
		if err == io.EOF {
			break
		}
		if err != nil && err != io.ErrUnexpectedEOF {
			return nil, fmt.Errorf("read chunk %d: %w", idx, err)
		}

		data := append([]byte(nil), buf[:n]...)
		sum := sha256.Sum256(data)
		digest := hex.EncodeToString(sum[:])
		clients, names := pickNodes(digest)

		var acks atomic.Int32
		g, gctx := errgroup.WithContext(ctx)
		for _, c := range clients {
			g.Go(func() error {
				if err := c.PutChunk(gctx, digest, data); err == nil {
					acks.Add(1)
				}
				return nil
			})
		}
		_ = g.Wait()

		if int(acks.Load()) < quorum {
			return nil, fmt.Errorf("chunk %s failed quorum (%d/%d)", digest, acks.Load(), quorum)
		}
		manifest = append(manifest, ChunkMeta{Index: idx, Digest: digest, Size: n, Nodes: names})
	}
	return manifest, nil
}
```

## Acceptance Drills

- Kill 1 of 3 worker nodes during an upload and verify quorum (`W=2`) still succeeds.
- Corrupt one chunk file on disk and verify SHA-256 verification detects it and falls back to a healthy replica.
- Export `storage_chunks_total`, `storage_replication_errors_total`, and `storage_node_up` to Prometheus.
