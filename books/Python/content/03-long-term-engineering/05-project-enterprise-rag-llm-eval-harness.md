---
title: '05. Project: Enterprise RAG/Agent Eval & Closed-to-Open LLM Migration Harness'
---

# Project: Enterprise RAG/Agent Eval & Closed-to-Open LLM Migration Harness

Build a Python evaluation and GRC audit pipeline that runs customer-specific golden datasets against both **closed frontier models** and **self-hosted open-weight models** (via vLLM / OpenAI-compatible APIs)—scoring tool-calling accuracy, JSON schema adherence, TTFT/TPOT latency, cost per 1M tokens, and PII leakage, blocking regressions in CI, and emitting **SHA-256 tamper-evident validation artifacts** for regulated enterprise sign-off.

## Evaluation Pipeline

```text
Golden Dataset (JSONL: prompt, expected_tool, expected_schema, forbidden_pii)
        |
        v
[ Python Async Eval Runner ]
   ├── Target A: Closed API (OpenAI / Anthropic)
   └── Target B: Open-Weight vLLM Endpoint (Llama / Qwen)
        |
        v
   Scorers: Schema Validator + Tool-Call F1 + PII Regex/Presidio + TTFT/TPOT + Cost
        |
        v
   CI Gate (exit 1 on regression) + Signed JSON/Markdown GRC Audit Report (SHA-256)
```

## Core Scoring & Tamper-Evident Artifact Generator

```python
import hashlib
import json
import re
import time
from dataclasses import asdict, dataclass

PII_PATTERNS = [
    re.compile(r"\b\d{3}-\d{2}-\d{4}\b"),  # SSN
    re.compile(r"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b"),
]

@dataclass
class EvalResult:
    case_id: str
    model: str
    schema_valid: bool
    tool_match: bool
    pii_leaked: bool
    ttft_ms: float
    cost_usd: float

def detect_pii(text: str) -> bool:
    return any(p.search(text) for p in PII_PATTERNS)

def write_audit_artifact(results: list[EvalResult], path: str) -> str:
    payload = {
        "generated_at": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
        "results": [asdict(r) for r in results],
    }
    raw = json.dumps(payload, sort_keys=True, indent=2).encode("utf-8")
    digest = hashlib.sha256(raw).hexdigest()
    with open(path, "wb") as f:
        f.write(raw)
    with open(path + ".sha256", "w", encoding="utf-8") as f:
        f.write(f"{digest}  {path}\n")
    return digest
```

## Acceptance Drills

- Run the harness against a baseline and a intentionally degraded prompt; verify CI fails on >3% tool-call accuracy drop or any PII leak (`pii_leaked == True`).
- Verify `sha256sum -c report.json.sha256` passes for every generated GRC artifact.
