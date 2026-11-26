---
title: '02. Function Calling — Structured LLM Actions'
---

# Function Calling — Structured LLM Actions

Free-form text responses are hard for application code to act on reliably. **Function calling (tool use)** lets you declare typed functions with JSON Schemas so the model emits structured, validated invocations that your Go backend executes in predictable, turn-based loops.

## Module Syllabus (6 Lessons)

1. Project Overview
2. Function Calling Intro
3. Using Function Calls in Projects Part 1
4. Using Function Calls in Projects Part 2
5. Implementing the Frontend
6. Adding Polish

## Turn-Based Function Calling Loop

```text
User Prompt + Tool Definitions
        |
        v
   [ LLM Call ]
        |
        +---> FinishReason: "stop"       --> Return final text/JSON to user
        |
        +---> FinishReason: "tool_calls" --> Validate args -> Execute Go function
                                                 |
                                                 v
                                       Append tool result message -> Loop back to LLM
```

The model never executes your Go code directly—it returns a function name and JSON arguments. Your Go service validates the arguments, executes the function, appends the result as a `tool` message, and sends the updated conversation back to the model.

## Defining and Registering Functions in Go

Maintain a clean registry that pairs each tool's JSON Schema with a typed Go handler.

```go
package tools

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
)

type Definition struct {
	Name        string         `json:"name"`
	Description string         `json:"description"`
	Parameters  map[string]any `json:"parameters"`
}

type HandlerFunc func(ctx context.Context, rawArgs json.RawMessage) (any, error)

type Registry struct {
	defs     []Definition
	handlers map[string]HandlerFunc
}

func NewRegistry() *Registry {
	return &Registry{handlers: make(map[string]HandlerFunc)}
}

func (r *Registry) Register(def Definition, fn HandlerFunc) {
	r.defs = append(r.defs, def)
	r.handlers[def.Name] = fn
}

func (r *Registry) Definitions() []Definition { return r.defs }

func (r *Registry) Execute(ctx context.Context, name string, rawArgs json.RawMessage) (string, error) {
	fn, ok := r.handlers[name]
	if !ok {
		return "", fmt.Errorf("unknown tool %q", name)
	}
	res, err := fn(ctx, rawArgs)
	if err != nil {
		return "", err
	}
	b, err := json.Marshal(res)
	if err != nil {
		return "", fmt.Errorf("marshal tool result: %w", err)
	}
	return string(b), nil
}

// Example: validated tool handler for saving a generated flashcard deck.
type SaveDeckArgs struct {
	Topic string   `json:"topic"`
	Cards []string `json:"cards"`
}

func SaveDeckHandler(_ context.Context, raw json.RawMessage) (any, error) {
	var args SaveDeckArgs
	if err := json.Unmarshal(raw, &args); err != nil {
		return nil, fmt.Errorf("invalid arguments: %w", err)
	}
	if args.Topic == "" || len(args.Cards) == 0 {
		return nil, errors.New("topic and at least one card are required")
	}
	return map[string]any{"status": "saved", "count": len(args.Cards)}, nil
}
```

## Validation & Frontend Polish

- **Validate every argument struct:** Treat LLM-produced JSON arguments with the same skepticism as external HTTP input. If validation fails, return a concise error message in the tool result so the model can self-correct on the next turn.
- **Cap turn counts:** Enforce a strict `maxTurns` limit (e.g., 5–8) to prevent infinite tool-call loops and runaway token bills.
- **Surface tool state in the frontend:** Show clear progress states ("Searching notes...", "Creating quiz...") and handle partial failures gracefully rather than hanging on a blank spinner.

## Related Chapters

- Previous: [01. Building Your First Production AI App](01-first-production-ai-app.md)
- Next: [03. Using Vector Databases](03-vector-databases.md)
- Also see: [04. Building AI Agents](04-building-ai-agents.md)
