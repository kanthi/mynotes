---
title: '05. Model Context Protocol'
---

# Model Context Protocol

Every AI app eventually needs to connect to external data sources and tools—databases, repositories, note stores, issue trackers. The **Model Context Protocol (MCP)** is an open JSON-RPC 2.0 standard that decouples AI hosts/agents from tool implementations, letting any MCP-compatible client discover and invoke tools exposed by any MCP server.

## Module Syllabus (5 Lessons)

1. How Does MCP work?
2. MCP servers in practice
3. Building MCP servers
4. Integrating MCP servers in your app
5. MCP best practices

## How MCP Works

```text
+----------------------------------+          JSON-RPC 2.0          +-----------------------------+
|  AI Host / Go Application        |  (stdio or Streamable HTTP)    |  MCP Server (Go / External) |
|  ├── LLM + Agent Loop            | <----------------------------> |  ├── Tools (actions)        |
|  └── MCP Client                  |   tools/list, tools/call       |  ├── Resources (read data)  |
|                                  |   resources/read, prompts/get  |  └── Prompts (templates)    |
+----------------------------------+                                +-----------------------------+
```

### Core Primitives

| Primitive | Controlled By | Purpose |
|-----------|---------------|---------|
| **Tools** | Model (with host approval) | Executable functions with JSON Schema inputs (e.g., `search_notes`, `create_flashcard`) |
| **Resources** | Application / User | Read-only URI-addressed context (e.g., `notes://go/concurrency.md`) |
| **Prompts** | User | Reusable parameterized prompt templates exposed by the server |

### Transports

- **`stdio`:** Local subprocess communication over `stdin`/`stdout` (ideal for local CLI tools and IDE integrations). Never write logs to `stdout` on a `stdio` server—log exclusively to `stderr`!
- **Streamable HTTP:** Remote network transport over HTTP/HTTPS with optional SSE streaming for shared team or cloud-hosted MCP servers.

## Building an MCP Server in Go

Using the official Go MCP SDK (`github.com/modelcontextprotocol/go-sdk/mcp`), you can expose typed Go functions as MCP tools with automatic schema generation.

```go
package main

import (
	"context"
	"log"
	"os"

	"github.com/modelcontextprotocol/go-sdk/mcp"
)

type SearchNotesInput struct {
	Query string `json:"query" jsonschema:"semantic search query over user notes"`
	Limit int    `json:"limit,omitempty" jsonschema:"maximum results to return (default 5)"`
}

type SearchNotesOutput struct {
	Matches []string `json:"matches"`
}

func main() {
	// Always log to stderr so stdout remains clean for JSON-RPC frames on stdio.
	log.SetOutput(os.Stderr)

	server := mcp.NewServer(&mcp.Implementation{
		Name:    "notes-mcp-server",
		Version: "1.0.0",
	}, nil)

	mcp.AddTool(server, &mcp.Tool{
		Name:        "search_notes",
		Description: "Search indexed study notes by topic or keyword.",
	}, func(ctx context.Context, _ *mcp.CallToolRequest, in SearchNotesInput) (*mcp.CallToolResult, SearchNotesOutput, error) {
		limit := in.Limit
		if limit <= 0 {
			limit = 5
		}
		// Delegate to your vector DB / notes service.
		matches := []string{"Example note matching: " + in.Query}
		if len(matches) > limit {
			matches = matches[:limit]
		}
		return nil, SearchNotesOutput{Matches: matches}, nil
	})

	if err := server.Run(context.Background(), &mcp.StdioTransport{}); err != nil {
		log.Fatalf("mcp server error: %v", err)
	}
}
```

## Integrating MCP Servers in Your Go App

Inside your Go agent application, connect an MCP client to one or more servers at startup, call `ListTools` to populate your agent's tool definitions dynamically, and route matching `ToolCall`s to `CallTool`. This lets you plug in new capabilities without recompiling or hard-coding every integration into your main app.

## MCP Best Practices

1. **Keep `stdout` pristine on `stdio` servers:** Any stray `fmt.Println` corrupts the JSON-RPC stream. Route all logs to `stderr`.
2. **Design narrow, least-privilege tools:** Scope database/API credentials per server, validate inputs strictly, and avoid exposing arbitrary SQL or shell execution tools to untrusted prompts.
3. **Return concise, LLM-friendly payloads:** Truncate large responses and return structured summaries instead of dumping megabytes of raw JSON into the model's context window.
4. **Enforce timeouts and cancellation:** Pass `ctx` through every tool handler so client disconnects cancel downstream database or HTTP calls immediately.
5. **Version and test tool contracts:** Write automated Go tests that exercise `tools/list` and `tools/call` against your MCP server just like any other RPC API.

## Related Chapters

- Previous: [04. Building AI Agents](04-building-ai-agents.md)
- Section Overview: [Ai-in-Go Intro](index.md)
- Also see: [ADK Go Agents](../28-specialized/95-adk-go-agents.md)
