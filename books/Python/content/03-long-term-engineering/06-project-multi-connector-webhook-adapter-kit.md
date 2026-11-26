---
title: '06. Project: Multi-Connector Enterprise Webhook & Legacy API Adapter Kit'
---

# Project: Multi-Connector Enterprise Webhook & Legacy API Adapter Kit

Forward Deployed Engineers frequently integrate messy enterprise systems (ITSM, CAD/GIS, ERP CSV/XML exports) into modern event-driven platforms. Build a production Python (FastAPI + Pydantic) integration bridge that normalizes heterogeneous webhooks/schemas into typed canonical events, validates HMAC signatures, routes malformed payloads to a replayable **Dead-Letter Queue (DLQ)**, and ships with OpenTofu/Terraform deployment templates for customer VPCs.

## Architecture

```text
Legacy Systems (ServiceNow / Jira / ERP Webhooks / SFTP CSV)
        |  HMAC-SHA256 signature verification
        v
[ Connector Adapter Layer (per-vendor normalizer) ]
        |
        +---> Valid   -> Canonical Pydantic Event -> Kafka / SQS / Webhook Target
        +---> Invalid -> Dead-Letter Queue (raw payload + validation error) -> Replay CLI
```

## Canonical Normalization with Pydantic

```python
from datetime import datetime, timezone
from pydantic import BaseModel, Field, ValidationError

class CanonicalIncidentEvent(BaseModel):
    event_id: str
    source: str
    external_id: str
    severity: str = Field(pattern="^(sev1|sev2|sev3|sev4)$")
    summary: str
    occurred_at: datetime

def normalize_legacy_itsm(raw: dict) -> CanonicalIncidentEvent:
    sev_map = {"1": "sev1", "2": "sev2", "3": "sev3", "4": "sev4"}
    return CanonicalIncidentEvent(
        event_id=f"itsm-{raw['sys_id']}",
        source="legacy_itsm",
        external_id=str(raw["number"]),
        severity=sev_map.get(str(raw.get("priority")), "sev4"),
        summary=raw["short_description"].strip(),
        occurred_at=datetime.now(timezone.utc),
    )
```

## Acceptance Drills

- Replay 100 messy fixture payloads (missing fields, duplicate deliveries, invalid HMAC) and verify valid events publish idempotently while malformed payloads land in the DLQ with full diagnostic metadata.
- Fix a schema mapping bug and run the DLQ replay CLI to drain previously failed events cleanly.
