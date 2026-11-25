---
title: Intro
---

# Intro

This part is a practical Go curriculum for **building and shipping production AI applications in Go**—progressing from your first LLM API call and deployed flashcard app to structured function calling, vector-database-backed RAG, tool-using AI agents, and the Model Context Protocol (MCP).

## Curriculum Architecture & Progression

```text
01. First Production AI App (14 lessons)
    LLM client -> API scaffold -> Prompting -> Flashcards + Notes ingestion -> Streaming -> Deploy
        |
        v
02. Function Calling — Structured LLM Actions (6 lessons)
    Tool schemas -> Structured JSON validation -> Turn-based action loops -> Frontend & polish
        |
        v
03. Using Vector Databases (9 lessons)
    Embeddings -> Chunking & indexing notes -> Semantic search -> Quiz v2 RAG API -> Deploy
        |
        v
04. Building AI Agents (9 lessons)
    Agent loop -> Tool registry -> App integration -> Prompt refinement -> Custom tools -> Deploy
        |
        v
05. Model Context Protocol (5 lessons)
    MCP architecture -> MCP servers in practice -> Building Go MCP servers -> App integration & best practices
```

## Module Syllabus & Chapter Map

| # | Module | Lessons | Core Focus | Chapter |
|---|--------|---------|------------|---------|
| 01 | **Building Your First Production AI App** | 14 | First LLM call, flashcards API, prompting, frontend, note ingestion, streaming, production deploy | [01. Building Your First Production AI App](01-first-production-ai-app.md) |
| 02 | **Function Calling — Structured LLM Actions** | 6 | Defining & registering functions, output validation, turn-based AI loops, frontend polish | [02. Function Calling — Structured LLM Actions](02-function-calling.md) |
| 03 | **Using Vector Databases** | 9 | Embeddings, semantic search, indexing notes, RAG-backed Quiz v2 API, production deploy | [03. Using Vector Databases](03-vector-databases.md) |
| 04 | **Building AI Agents** | 9 | Tool-using agent loops, app & frontend integration, prompt tuning, custom tools, deploy | [04. Building AI Agents](04-building-ai-agents.md) |
| 05 | **Model Context Protocol** | 5 | MCP architecture, servers in practice, building & integrating Go MCP servers, best practices | [05. Model Context Protocol](05-model-context-protocol.md) |

## Related Parts in This Book

- [Building LLM Applications in Go](../28-specialized/91-llm-apps.md) — provider clients, structured outputs, and production LLM guardrails.
- [ADK Go Agents](../28-specialized/95-adk-go-agents.md) — building multi-agent workflows with Google's Agent Development Kit for Go.
- [Web Development in Go](../15-web-development-in-go/270-web-dev-golang-index.md) — HTTP routing, templates, databases, and graceful shutdown.
- [Cloud and Infrastructure](../20-infrastructure/70-docker-multistage.md) — multi-stage Docker builds, deployment, and production operations.
