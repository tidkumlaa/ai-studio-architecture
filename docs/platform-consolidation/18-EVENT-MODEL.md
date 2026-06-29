---
knowledge_id: KNW-PLAT-ARCH-018
title: "Platform Event Model"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Define all platform events, schemas, routing, and cross-runtime communication patterns"
canonical_source: "architecture/docs/platform-consolidation/18-EVENT-MODEL.md"
dependencies:
  - "07-KERNEL-MODEL.md"
  - "06-RUNTIME-MODEL.md"
  - "10-DEPENDENCY-RULES.md"
related_documents:
  - "19-STATE-MACHINES.md"
  - "32-RUNTIME-INTEGRATION.md"
acceptance_criteria:
  - "Every cross-runtime event has a schema"
  - "All event topics follow naming convention"
  - "No direct function calls between runtimes"
verification_checklist:
  - "[ ] All event schemas are Pydantic models"
  - "[ ] Event topics follow naming convention"
  - "[ ] Routing rules documented"
future_extensions:
  - "Event replay for debugging"
  - "Event sourcing for audit trail"
---

# Platform Event Model

## Event Communication Principle

Cross-runtime communication uses **events only**. No runtime calls another runtime's
functions directly. All coordination passes through the `EventBus`.

```
runtime:A  →  EventBus.publish(event)  →  EventBus  →  runtime:B.handle(event)
```

See DR-002 in 10-DEPENDENCY-RULES.md.

---

## Event Schema (Base)

```python
from __future__ import annotations
from dataclasses import dataclass, field
from datetime import datetime
import uuid

@dataclass(frozen=True)
class PlatformEvent:
    topic: str                        # "platform.{domain}.{name}"
    event_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    occurred_at: datetime = field(default_factory=datetime.utcnow)
    source_runtime: str = ""          # "runtime:ai:v2"
    correlation_id: str = ""          # for tracing request flows
    payload: dict = field(default_factory=dict)
```

---

## Topic Naming Convention

```
platform.{domain}.{verb}[.{qualifier}]
```

| Part | Rules |
|------|-------|
| `platform` | Literal prefix, always present |
| `domain` | Runtime short name: `ai`, `resource`, `provider`, `knowledge`, `workflow`, `orchestration`, `kernel` |
| `verb` | Past tense action: `started`, `completed`, `failed`, `requested`, `updated`, `registered` |
| `qualifier` | Optional sub-topic, e.g., `.quota`, `.usage`, `.health` |

Examples:
- `platform.ai.execution.completed`
- `platform.provider.health.degraded`
- `platform.resource.quota.exceeded`
- `platform.knowledge.compile.completed`

---

## Kernel Events

| Topic | Trigger | Payload Fields |
|-------|---------|----------------|
| `platform.kernel.runtime.started` | Runtime boot complete | `runtime_id`, `boot_order`, `duration_ms` |
| `platform.kernel.runtime.stopped` | Runtime shutdown | `runtime_id`, `reason` |
| `platform.kernel.runtime.failed` | Runtime boot failed | `runtime_id`, `error`, `stack_trace` |
| `platform.kernel.config.reloaded` | Config hot-reload | `changed_keys: list[str]` |

---

## AI Runtime Events (Published)

| Topic | Trigger | Payload Fields |
|-------|---------|----------------|
| `platform.ai.quota.consumed` | Token quota consumed | `account_id`, `model_id`, `tokens_used`, `remaining` |
| `platform.ai.quota.exceeded` | Quota limit hit | `account_id`, `model_id`, `limit`, `requested` |
| `platform.ai.budget.exceeded` | Cost budget hit | `account_id`, `cost_usd`, `limit_usd` |
| `platform.ai.execution.planned` | Execution plan created | `plan_id`, `strategy`, `agent_count` |
| `platform.ai.execution.completed` | Task execution done | `plan_id`, `tokens_used`, `cost_usd`, `latency_ms` |
| `platform.ai.execution.failed` | Execution failure | `plan_id`, `error`, `retry_count` |
| `platform.ai.recommendation.issued` | Recommendation created | `recommendation_id`, `action`, `reason` |
| `platform.ai.learning.recorded` | Prediction outcome recorded | `prediction_id`, `actual_tokens`, `accuracy` |

---

## AI Runtime Events (Subscribed)

| Topic | Handler | Action |
|-------|---------|--------|
| `platform.provider.health.degraded` | `ai_runtime.on_provider_health` | Adjust routing weights |
| `platform.resource.quota.exceeded` | `ai_runtime.on_quota_exceeded` | Block new tasks for account |
| `platform.kernel.config.reloaded` | `ai_runtime.on_config_reload` | Refresh pricing and model catalog |

---

## Resource Runtime Events (Published)

| Topic | Trigger | Payload Fields |
|-------|---------|----------------|
| `platform.resource.quota.updated` | Quota limit changed | `account_id`, `old_limit`, `new_limit` |
| `platform.resource.budget.updated` | Budget limit changed | `account_id`, `old_budget`, `new_budget` |
| `platform.resource.schedule.queued` | Job queued | `job_id`, `priority`, `estimated_start` |

---

## Provider Runtime Events (Published)

| Topic | Trigger | Payload Fields |
|-------|---------|----------------|
| `platform.provider.registered` | New provider registered | `provider_id`, `capabilities` |
| `platform.provider.health.degraded` | Provider latency/error spike | `provider_id`, `error_rate`, `p99_latency_ms` |
| `platform.provider.health.restored` | Provider recovered | `provider_id`, `recovered_at` |

---

## Knowledge Runtime Events (Published)

| Topic | Trigger | Payload Fields |
|-------|---------|----------------|
| `platform.knowledge.compile.completed` | Document compiled | `knowledge_id`, `title`, `duration_ms` |
| `platform.knowledge.index.updated` | Index rebuilt | `entry_count`, `duration_ms` |
| `platform.knowledge.link.created` | Graph edge added | `from_id`, `to_id`, `relation` |

---

## EventBus Protocol

```python
from typing import Protocol, Callable, Awaitable

class EventBus(Protocol):
    async def publish(self, event: PlatformEvent) -> None: ...

    def subscribe(
        self,
        topic_pattern: str,
        handler: Callable[[PlatformEvent], Awaitable[None]],
    ) -> str: ...  # returns subscription_id

    def unsubscribe(self, subscription_id: str) -> None: ...

    async def stream(
        self,
        topic_pattern: str,
    ) -> AsyncIterator[PlatformEvent]: ...
```

Topic pattern supports wildcards:
- `platform.ai.*` — all AI runtime events
- `platform.*.health.*` — all health events across runtimes
- `platform.#` — all platform events

---

## Event Routing Rules

| Rule | Description |
|------|-------------|
| ER-001 | Events are fire-and-forget. Publishers do not wait for subscriber response. |
| ER-002 | Subscribers must not raise exceptions that crash the bus. Exceptions are logged and the handler is skipped. |
| ER-003 | A handler must complete within 5 seconds or be cancelled. |
| ER-004 | Events with no subscribers are silently dropped (no error). |
| ER-005 | Events published during runtime shutdown are dropped after all handlers drain. |
| ER-006 | Cross-runtime handlers run in the subscriber runtime's executor, never the publisher's. |

---

## Event Payload Size Limits

| Limit | Value |
|-------|-------|
| Max payload size | 64 KB |
| Max topic length | 256 characters |
| Max subscribers per topic | 50 |
| Handler timeout | 5 seconds |
