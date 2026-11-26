---
title: '03. Using Vector Databases'
---

# Using Vector Databases

Context windows are finite and expensive. **Vector databases** store dense numerical embeddings of your notes so you can run fast semantic similarity search and inject only the most relevant chunks into each LLM request (Retrieval-Augmented Generation, or RAG).

## Module Syllabus (9 Lessons)

1. Project Intro
2. Introduction to Vector DBs
3. Vector DB Example and Demo
4. Vector DB Crash Course
5. Quiz v2 - API Scaffold
6. Index Notes
7. Implementing Our New Quiz API
8. Frontend and Polish
9. Deploying to Production

## RAG Indexing & Query Pipeline

```text
Ingestion (Offline / On-Upload):
  Raw Notes -> Chunking (overlap) -> Embedding Model -> Upsert (vector + text + metadata) -> Vector DB

Query Time (Quiz v2 API):
  User Topic/Query -> Embedding Model -> Top-K Similarity Search (Vector DB)
                                                |
                                                v
                                     Retrieved Note Chunks + Quiz Prompt -> LLM -> Quiz Questions
```

## Vector DB Concepts (Crash Course)

| Concept | Role |
|---------|------|
| **Embedding** | Fixed-length `[]float32` vector (e.g., 1536 dims) capturing semantic meaning |
| **Similarity Metric** | Cosine distance (`<=>`), inner product, or L2 distance to rank nearest neighbors |
| **Chunking** | Splitting long notes into ~300–800 token passages with small overlap (~10–15%) so facts are not severed at boundaries |
| **Metadata Filtering** | Filtering by `user_id`, `notebook`, or `tag` *before* or *during* vector search to enforce multi-tenant isolation |
| **ANN Indexes** | HNSW or IVFFlat indexes for sub-linear approximate nearest-neighbor lookup at scale |

## Indexing Notes and Querying in Go (`pgvector` / PostgreSQL)

PostgreSQL with `pgvector` keeps relational metadata and vector embeddings in a single operational database, while dedicated stores like Qdrant or Weaviate follow the same `Upsert` / `Search` interface.

```go
package rag

import (
	"context"
	"database/sql"
	"fmt"
	"strings"
)

type Chunk struct {
	ID      string
	NoteID  string
	Content string
	Vector  []float32
}

type Embedder interface {
	Embed(ctx context.Context, text string) ([]float32, error)
}

// ChunkText splits a note into overlapping word-bounded windows.
func ChunkText(text string, maxWords, overlap int) []string {
	words := strings.Fields(text)
	if len(words) == 0 {
		return nil
	}
	var chunks []string
	step := max(1, maxWords-overlap)
	for start := 0; start < len(words); start += step {
		end := min(len(words), start+maxWords)
		chunks = append(chunks, strings.Join(words[start:end], " "))
		if end == len(words) {
			break
		}
	}
	return chunks
}

// SearchTopK retrieves the k most semantically similar note chunks for a user.
func SearchTopK(ctx context.Context, db *sql.DB, userID string, queryVec string, k int) ([]string, error) {
	const q = `
		SELECT content
		FROM note_chunks
		WHERE user_id = $1
		ORDER BY embedding <=> $2::vector
		LIMIT $3
	`
	rows, err := db.QueryContext(ctx, q, userID, queryVec, k)
	if err != nil {
		return nil, fmt.Errorf("vector search: %w", err)
	}
	defer rows.Close()

	var results []string
	for rows.Next() {
		var content string
		if err := rows.Scan(&content); err != nil {
			return nil, err
		}
		results = append(results, content)
	}
	return results, rows.Err()
}
```

## Common Pitfalls

- **Mismatching embedding models:** You must use the exact same embedding model and dimensionality for indexing and querying. Changing models requires re-indexing all notes.
- **Missing tenant filters:** Always scope similarity queries by `user_id` or `tenant_id` so one user's quiz never retrieves another user's private notes.
- **Re-indexing everything on every edit:** Track a content hash per note so unchanged notes are skipped during ingestion.

## Related Chapters

- Previous: [02. Function Calling — Structured LLM Actions](02-function-calling.md)
- Next: [04. Building AI Agents](04-building-ai-agents.md)
- Also see: [PostgreSQL and Redis Integrations](../16-web/64-db-postgres-redis.md)
