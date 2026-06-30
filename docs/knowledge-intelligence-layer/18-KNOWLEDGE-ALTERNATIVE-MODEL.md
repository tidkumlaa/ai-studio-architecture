# KNW-KIL-DOC-018 — Knowledge Alternative Model

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

For every Knowledge Object that represents a specific design choice, the
Alternative Model records what else could have been built instead — different
architectures, different algorithms, different designs that would satisfy similar
requirements.

This serves two purposes:
1. Helps AI agents avoid confusing this object with its alternatives
2. Provides architects with a pre-researched option space for future re-evaluation

This document defines `intelligence.alternatives`.

---

## Schema

```yaml
intelligence:
  alternatives:
    - alternative_id: string           # e.g. ALT-001
      name: string
      knowledge_id: string | null      # if this alternative is a KOS object
      status: REJECTED | DEFERRED | SUPERSEDED | PARALLEL | FUTURE

      description: string              # what this alternative is
      similarity_to_this: float        # 0.0–1.0 semantic similarity

      would_satisfy:                   # what requirements it could fulfill
        - knowledge_id: string
          completeness: FULL | PARTIAL

      pros:                            # why this alternative has merit
        - dimension: string
          description: string

      cons:                            # why this alternative was not chosen
        - dimension: string
          description: string

      rejection_reason: string | null  # why it was rejected (if status: REJECTED)
      deferral_reason: string | null   # why it was deferred (if status: DEFERRED)
      evaluation_date: string          # when this alternative was evaluated
      evaluated_by: string

      conditions_for_reconsideration: [string]
```

---

## Alternative Status

| Status | Meaning |
|--------|---------|
| REJECTED | Evaluated and deliberately not chosen |
| DEFERRED | Could be chosen in the future; not ruled out |
| SUPERSEDED | Was the previous choice; replaced by this object |
| PARALLEL | Coexists with this object for different use cases |
| FUTURE | Anticipated but not yet evaluated |

---

## Canonical Example

```yaml
# For: KNW-PLT-MOD-001 (Quota Manager)
intelligence:
  alternatives:
    - alternative_id: ALT-001
      name: "Sidecar Quota Enforcer"
      knowledge_id: null
      status: REJECTED
      description: >
        A sidecar proxy pattern where quota enforcement runs as a network
        interceptor rather than an in-process module. Each service would
        communicate through a quota-aware proxy.
      similarity_to_this: 0.55

      would_satisfy:
        - knowledge_id: KNW-PLT-REQ-001
          completeness: FULL

      pros:
        - dimension: "language-agnostic"
          description: "Works with any service regardless of implementation language"
        - dimension: "independent-scaling"
          description: "Quota enforcement scales independently from services"

      cons:
        - dimension: "latency"
          description: "Network hop adds 5–20ms overhead (violates P99 < 5ms)"
        - dimension: "complexity"
          description: "Requires service mesh infrastructure"
        - dimension: "operational-overhead"
          description: "Sidecar must be deployed and managed alongside every service"

      rejection_reason: >
        Network latency overhead (5–20ms) violates the P99 < 5ms requirement
        (KNW-META-STD-007). Rejected at architecture review 2025-01-08.
      deferral_reason: null
      evaluation_date: "2025-01-08"
      evaluated_by: "architecture-board"
      conditions_for_reconsideration:
        - "P99 requirement relaxed to > 10ms"
        - "Service mesh (e.g., Istio) adopted platform-wide"

    - alternative_id: ALT-002
      name: "Database-backed Quota Counter"
      knowledge_id: null
      status: REJECTED
      description: >
        Store quota state in a relational database, queried on every request.
        Uses row-level locking to prevent over-allowance.
      similarity_to_this: 0.60

      would_satisfy:
        - knowledge_id: KNW-PLT-REQ-001
          completeness: FULL

      pros:
        - dimension: "durability"
          description: "Quota state survives restarts without registry dependency"
        - dimension: "familiarity"
          description: "SQL patterns well-understood by most engineers"

      cons:
        - dimension: "latency"
          description: "Database round-trip is 10–50ms — violates P99 < 5ms"
        - dimension: "scalability"
          description: "Database becomes bottleneck under high concurrency"
        - dimension: "locking"
          description: "Row-level locking creates contention at scale"

      rejection_reason: >
        Database round-trip latency (10–50ms) violates P99 < 5ms. SQL locking
        creates contention at the required throughput (> 10K req/s).
      deferral_reason: null
      evaluation_date: "2025-01-09"
      evaluated_by: "platform-team"
      conditions_for_reconsideration:
        - "P99 requirement > 50ms for quota-gated endpoints"
        - "In-memory database (Redis) used instead of relational"

    - alternative_id: ALT-003
      name: "KNW-ALG-ALG-009 Rate Limiter"
      knowledge_id: KNW-ALG-ALG-009
      status: PARALLEL
      description: >
        Token Bucket rate limiter controls request velocity per window.
        Often confused with quota enforcement but serves a different purpose.
      similarity_to_this: 0.30

      would_satisfy:
        - knowledge_id: KNW-PLT-REQ-001
          completeness: PARTIAL

      pros:
        - dimension: "velocity control"
          description: "Prevents request bursts more effectively than quota"
        - dimension: "widely understood"
          description: "Token bucket is a standard algorithm"

      cons:
        - dimension: "wrong-dimension"
          description: "Controls req/s, not cumulative resource consumption"
        - dimension: "partial-coverage"
          description: "Does not enforce CPU/Memory/Storage quotas"

      rejection_reason: null
      deferral_reason: null
      evaluation_date: "2025-01-10"
      evaluated_by: "platform-team"
      conditions_for_reconsideration: []

    - alternative_id: ALT-004
      name: "AI-driven Dynamic Quota Allocation"
      knowledge_id: null
      status: FUTURE
      description: >
        An AI model dynamically adjusts quota limits based on predicted usage
        patterns, historical demand, and fairness constraints.
      similarity_to_this: 0.40

      would_satisfy:
        - knowledge_id: KNW-PLT-REQ-001
          completeness: PARTIAL

      pros:
        - dimension: "efficiency"
          description: "Better resource utilization across tenants"
        - dimension: "adaptivity"
          description: "Responds to usage patterns without manual tuning"

      cons:
        - dimension: "complexity"
          description: "Requires ML model training and serving infrastructure"
        - dimension: "explainability"
          description: "Dynamic limits may be hard for tenants to predict"

      rejection_reason: null
      deferral_reason: "Requires AI Runtime (Phase 3.0E) to be complete first"
      evaluation_date: "2025-01-10"
      evaluated_by: "architecture-board"
      conditions_for_reconsideration:
        - "AI Runtime (Phase 3.0E) complete and stable"
        - "Tenant feedback indicates static quotas are insufficient"
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KIL-103 | Every object with `semantic.conflicts` must have a corresponding `alternative` entry |
| KIL-104 | Every `status: REJECTED` alternative must have a non-null `rejection_reason` |
| KIL-105 | `conditions_for_reconsideration` must be defined for every REJECTED alternative |
| KIL-106 | `status: PARALLEL` alternatives must have `knowledge_id` set if they exist as KOS objects |
| KIL-107 | `similarity_to_this > 0.70` alternatives require explicit differentiation in `ai_context.typical_mistakes` |

---

## Cross-References

- Tradeoff model → `16-KNOWLEDGE-TRADEOFF-MODEL`
- Decision model → `17-KNOWLEDGE-DECISION-MODEL`
- AI context typical mistakes → `04-AI-CONTEXT-LAYER`
- Semantic conflicts → `03-SEMANTIC-KNOWLEDGE`
