---
title: 'Project: Service Test Suite'
---

# Project: Service Test Suite

Design a layered test suite for a small HTTP + repository Go service that combines fast **table-driven unit tests**, **HTTP handler tests** (`httptest`), **fuzzing** for input parsers, and **race-checked concurrency tests**.

## Test Pyramid for a Go Service

```text
          /\
         /  \     Component / Integration (real DB or testcontainers, -tags=integration)
        /----\
       /      \   HTTP Handler Tests (httptest.NewRecorder + in-memory fake repo)
      /--------\
     /          \ Unit + Fuzz Tests (pure domain validation, parsers, edge cases)
    /____________\
```

## Complete Test Suite Example

```go
package service_test

import (
	"bytes"
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"sync"
	"testing"
)

type Item struct {
	ID   string `json:"id"`
	Name string `json:"name"`
}

type FakeStore struct {
	mu    sync.RWMutex
	items map[string]Item
}

func NewFakeStore() *FakeStore { return &FakeStore{items: make(map[string]Item)} }

func (s *FakeStore) Put(it Item) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.items[it.ID] = it
}

func (s *FakeStore) Get(id string) (Item, bool) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	it, ok := s.items[id]
	return it, ok
}

func newHandler(store *FakeStore) http.Handler {
	mux := http.NewServeMux()
	mux.HandleFunc("POST /items", func(w http.ResponseWriter, r *http.Request) {
		var it Item
		if err := json.NewDecoder(r.Body).Decode(&it); err != nil || it.ID == "" || it.Name == "" {
			http.Error(w, "invalid item", http.StatusBadRequest)
			return
		}
		store.Put(it)
		w.WriteHeader(http.StatusCreated)
	})
	mux.HandleFunc("GET /items/{id}", func(w http.ResponseWriter, r *http.Request) {
		it, ok := store.Get(r.PathValue("id"))
		if !ok {
			http.NotFound(w, r)
			return
		}
		_ = json.NewEncoder(w).Encode(it)
	})
	return mux
}

func TestItemHTTPFlow(t *testing.T) {
	t.Parallel()
	h := newHandler(NewFakeStore())

	tests := []struct {
		name       string
		method     string
		path       string
		body       string
		wantStatus int
	}{
		{"create valid", http.MethodPost, "/items", `{"id":"1","name":"book"}`, http.StatusCreated},
		{"create invalid", http.MethodPost, "/items", `{"id":""}`, http.StatusBadRequest},
		{"get existing", http.MethodGet, "/items/1", "", http.StatusOK},
		{"get missing", http.MethodGet, "/items/999", "", http.StatusNotFound},
	}

	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			req := httptest.NewRequest(tc.method, tc.path, bytes.NewBufferString(tc.body))
			rec := httptest.NewRecorder()
			h.ServeHTTP(rec, req)
			if rec.Code != tc.wantStatus {
				t.Fatalf("status = %d, want %d (body=%q)", rec.Code, tc.wantStatus, rec.Body.String())
			}
		})
	}
}
```

## Verification Commands

```bash
go test -v -race -cover ./...
go test -fuzz=Fuzz -fuzztime=10s ./...
```
