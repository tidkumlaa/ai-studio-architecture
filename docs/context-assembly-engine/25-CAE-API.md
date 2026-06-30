# CAE-DOC-025 — CAE API

**Phase:** 3.0D.1
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The CAE API defines the external contract of the Context Assembly Engine —
the precise inputs, outputs, and error types that any implementation must satisfy.
The API is synchronous from the caller's perspective: `assemble(request) → pack`.

---

## Primary Assembly API

### Request

```
AssemblyRequest:
  query:    QueryRecord           # required
  consumer: ConsumerProfile       # required
  budget:   BudgetSpec            # required

QueryRecord:
  query_id:   string              # caller-supplied unique identifier
  raw_query:  string              # natural language or structured query
  query_type: string | null       # optional; inferred by S1 if null
  entities:   [string] | null     # optional pre-resolved entity IDs

ConsumerProfile:
  type:     AI_AGENT | DEVELOPER | ARCHITECT | EXECUTIVE | OPERATOR | SEARCH_UI
  audience: string | null         # DEVELOPER | ARCHITECT | EXECUTIVE (for HumanPack)
  hints:    {string: string}      # optional; passed to intent parser

BudgetSpec:
  total_tokens: integer           # required; 100–20,000
  pack_type: string | null        # optional; inferred from consumer.type if null
```

### Response

```
AssemblyResponse:
  pack:         ContextPack       # one of: PromptPack, HumanPack, OperatorPack, SearchPack
  request_id:   string            # echoes query.query_id
  assembly_id:  string            # internal UUID for this assembly
  latency_ms:   integer           # total E2E latency
```

---

## ContextPack Union Type

```
ContextPack = PromptPack | HumanPack | OperatorPack | SearchPack

All ContextPack types share:
  pack_id:       string
  pack_type:     string
  query_id:      string
  generated_at:  string
  pack_metadata: PackMetadata

PackMetadata:
  total_tokens:       integer
  budget_declared:    integer
  budget_utilization: float
  context_confidence: float
  assembly_latency_ms: integer
  objects_included:   integer
  objects_dropped:    integer
  quality_report:     QualityReport

QualityReport:
  context_quality_score: float
  quality_level:         string
  components:
    relevance:     float
    completeness:  float
    safety:        float
    efficiency:    float
  improvement_hints: [string]
```

---

## Error Types

```
AssemblyError:
  error_code:  string
  message:     string
  request_id:  string
  details:     object | null

Error Codes:

  ENTITY_NOT_FOUND
    message: "No object in the KIL index matches the query entities."
    details: {entities_tried: [string]}

  AIRS_TOO_LOW
    message: "{knowledge_id} AIRS={score} below 0.60 minimum for PromptPack."
    details: {knowledge_id: string, airs: float}

  BUDGET_TOO_SMALL
    message: "Minimum assembly requires {min_tokens} tokens; budget is {budget} tokens."
    details: {min_tokens: integer, budget: integer}

  QUALITY_TOO_LOW
    message: "Context quality {score:.2f} below minimum 0.70."
    details: {quality_score: float, components: QualityComponents}

  VALIDATION_FAILED
    message: "Pack failed validation."
    details: {errors: [ValidationError]}

  LATENCY_BUDGET_EXCEEDED
    message: "CAE latency exceeded 200ms during emergency mode — request rejected."
    details: {latency_ms: integer}

  INVALID_REQUEST
    message: "Request validation failed."
    details: {field: string, reason: string}
```

---

## Feedback API

### Submit Feedback

```
FeedbackRequest:
  pack_id:    string              # required
  query_id:   string              # required
  source:     EXPLICIT | IMPLICIT # required

  # For EXPLICIT feedback
  explicit:
    rating:              integer | null    # 1–5 (null for AI agents)
    useful:              boolean | null
    missing_info:        [string]
    misleading:          [string]
    hallucination_flagged: boolean

  # For IMPLICIT feedback
  implicit:
    signal_type:         RE_QUERY | RAPID_FOLLOW_UP | ABANDONED | USED
    latency_to_signal_ms: integer
    follow_up_query_id:  string | null

FeedbackResponse:
  feedback_id: string
  accepted:    boolean
```

---

## Search API (Thin Wrapper)

```
SearchRequest:
  query_text:    string           # required
  filters:
    types:       [string]         # filter by object_type
    namespaces:  [string]
    min_quality: float            # default: 0.70
    min_airs:    float            # default: 0.60
  limit:         integer          # 1–20; default: 10
  budget_tokens: integer          # default: 500

SearchResponse:
  pack: SearchPack                # see doc 14
```

---

## API Design Constraints

```
1. Synchronous: assemble() must return in < P99=120ms
   Callers must not need to poll

2. Stateless: no session state between calls
   All context must be in the request

3. Idempotent read: same (query, consumer, budget) → same pack structure
   (pack_id and generated_at differ; content is deterministic)

4. Error transparency: every AssemblyError carries enough detail to diagnose
   the root cause — no opaque failures

5. Budget strict: the response pack ALWAYS fits within total_tokens
   An over-budget response is a bug, not a warning
```

---

## Versioning

```
API version: v1
Version is carried in:
  Request header: X-CAE-API-Version: v1
  Response header: X-CAE-API-Version: v1

Breaking changes require new version (v2):
  - Removing a required field
  - Changing error codes
  - Changing QualityReport formula

Non-breaking changes (v1 compatible):
  - Adding optional fields
  - New pack types
  - New error codes with new error_code strings
```

---

## Rules

| Rule | Statement |
|------|-----------|
| CAE-116 | API contract is stable after CAE architecture freeze — no breaking changes in v1 |
| CAE-117 | AssemblyError must always include request_id for correlation |
| CAE-118 | Feedback API must accept both EXPLICIT and IMPLICIT sources — both are first-class |
| CAE-119 | Idempotency: same effective inputs must produce structurally identical packs |
| CAE-120 | Budget constraint is guaranteed by the API — callers must never need to check |

---

## Cross-References

- All pack builders → docs 11–14
- Context validator → `16-CONTEXT-VALIDATOR`
- Feedback model → `22-FEEDBACK-MODEL`
- CAE certification → `26-CAE-CERTIFICATION`
- CAE testing → `27-CAE-TESTING`
