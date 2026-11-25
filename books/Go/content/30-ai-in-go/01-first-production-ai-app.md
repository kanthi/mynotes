---
title: '01. Building Your First Production AI App'
---

# Building Your First Production AI App

Go from your first LLM API call to a deployed, end-to-end **AI Flashcards** application—covering environment setup, project scaffolding, prompt iteration, real-note ingestion, frontend integration, production deployment, and token streaming.

## Module Syllabus (14 Lessons)

1. Project Overview
2. Setting Up Your Development Environment
3. Initializing Project From Template and Deploying
4. Introducing the Project and Building Basic Flashcards App
5. Making the First LLM Call
6. Adding AI to Our Project
7. Scaffolding our API to make LLM Calls
8. Improving our Prompts
9. Building the Frontend
10. Deploying the Frontend
11. Ingesting Real Notes
12. Polishing the Project
13. Wrap Up and Challenges
14. BONUS: LLM Streaming

## Application Architecture

```text
Browser / Frontend
   |
   |  POST /api/flashcards (note text or uploaded markdown)
   v
Go HTTP API (net/http)
   ├── Validate input & enforce size/rate limits
   ├── Prompt Builder (system instructions + user notes + output format)
   ├── LLM Client (timeouts, retries, structured JSON parsing)
   └── Persistence / Response (JSON or SSE token stream)
```

Deploy early (lesson 3) with a minimal health endpoint and template scaffold so CI/CD, environment variables (`LLM_API_KEY`), and TLS are proven before adding AI complexity.

## Making the First LLM Call & Scaffolding the API

Wrap the LLM provider behind a small Go interface so handlers stay testable and provider-agnostic.

```go
package ai

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"net/http"
	"time"
)

type Flashcard struct {
	Question string `json:"question"`
	Answer   string `json:"answer"`
}

type Generator interface {
	GenerateFlashcards(ctx context.Context, notes string) ([]Flashcard, error)
}

type Client struct {
	apiKey     string
	baseURL    string
	model      string
	httpClient *http.Client
}

func NewClient(apiKey, baseURL, model string) *Client {
	return &Client{
		apiKey:  apiKey,
		baseURL: baseURL,
		model:   model,
		httpClient: &http.Client{
			Timeout: 45 * time.Second,
		},
	}
}

func (c *Client) GenerateFlashcards(ctx context.Context, notes string) ([]Flashcard, error) {
	body := map[string]any{
		"model": c.model,
		"messages": []map[string]string{
			{
				"role": "system",
				"content": "You generate concise study flashcards from technical notes. " +
					"Return valid JSON only: {\"flashcards\":[{\"question\":\"...\",\"answer\":\"...\"}]}",
			},
			{
				"role":    "user",
				"content": notes,
			},
		},
		"response_format": map[string]string{"type": "json_object"},
	}

	raw, err := json.Marshal(body)
	if err != nil {
		return nil, fmt.Errorf("marshal request: %w", err)
	}

	req, err := http.NewRequestWithContext(ctx, http.MethodPost, c.baseURL+"/chat/completions", bytes.NewReader(raw))
	if err != nil {
		return nil, fmt.Errorf("create request: %w", err)
	}
	req.Header.Set("Authorization", "Bearer "+c.apiKey)
	req.Header.Set("Content-Type", "application/json")

	resp, err := c.httpClient.Do(req)
	if err != nil {
		return nil, fmt.Errorf("llm request: %w", err)
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return nil, fmt.Errorf("llm status %d", resp.StatusCode)
	}

	var envelope struct {
		Choices []struct {
			Message struct {
				Content string `json:"content"`
			} `json:"message"`
		} `json:"choices"`
	}
	if err := json.NewDecoder(resp.Body).Decode(&envelope); err != nil {
		return nil, fmt.Errorf("decode envelope: %w", err)
	}
	if len(envelope.Choices) == 0 {
		return nil, fmt.Errorf("empty llm response")
	}

	var out struct {
		Flashcards []Flashcard `json:"flashcards"`
	}
	if err := json.Unmarshal([]byte(envelope.Choices[0].Message.Content), &out); err != nil {
		return nil, fmt.Errorf("decode flashcards json: %w", err)
	}
	return out.Flashcards, nil
}
```

## Improving Prompts & Ingesting Real Notes

- **Separate instructions from untrusted user content:** Place rules in the `system` message and delimit raw user notes clearly so markdown headings or stray instructions inside notes do not hijack the prompt.
- **Specify card quality constraints:** Constrain card count, target atomic one-concept questions, require answers grounded strictly in the provided notes, and ban filler phrasing.
- **Normalize ingested notes:** Strip binary/front-matter noise, enforce a maximum token/byte budget (e.g., `http.MaxBytesReader`), and split oversized documents into section-aware chunks before calling the model.

## BONUS: LLM Streaming with Server-Sent Events (SSE)

Long generations feel slow when buffered until completion. Stream tokens from the Go API to the frontend using `http.Flusher` and `text/event-stream`.

```go
func StreamHandler(w http.ResponseWriter, r *http.Request, tokens <-chan string) {
	flusher, ok := w.(http.Flusher)
	if !ok {
		http.Error(w, "streaming unsupported", http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "text/event-stream")
	w.Header().Set("Cache-Control", "no-cache")
	w.Header().Set("Connection", "keep-alive")

	for {
		select {
		case <-r.Context().Done():
			return
		case tok, open := <-tokens:
			if !open {
				fmt.Fprintf(w, "event: done\ndata: {}\n\n")
				flusher.Flush()
				return
			}
			fmt.Fprintf(w, "data: %s\n\n", tok)
			flusher.Flush()
		}
	}
}
```

## Common Pitfalls

- **No timeout on LLM HTTP calls:** Model endpoints occasionally stall for minutes; always use `context.WithTimeout` and explicit `http.Client.Timeout`.
- **Exposing API keys to the frontend:** Never call LLM providers directly from browser JavaScript—proxy all calls through your Go backend where you control secrets, quotas, and prompt templates.
- **Trusting raw LLM output without validation:** Even with JSON mode enabled, validate required fields and lengths before persisting or rendering.

## Related Chapters

- Next: [02. Function Calling — Structured LLM Actions](02-function-calling.md)
- Also see: [Building LLM Applications in Go](../28-specialized/91-llm-apps.md), [Docker Multi-Stage Builds](../20-infrastructure/70-docker-multistage.md)
