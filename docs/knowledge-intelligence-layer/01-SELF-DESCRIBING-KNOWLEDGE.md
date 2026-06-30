# KNW-KIL-DOC-001 — Self-Describing Knowledge

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Every Knowledge Object must be capable of describing itself completely to any
consumer — human, AI agent, or automated system — without requiring external
context.

This document defines the `intelligence.self_describing` block.

---

## Schema

```yaml
intelligence:
  self_describing:
    purpose: string                    # one sentence: why does this object exist?
    responsibilities:                  # list: what does it own / guarantee?
      - string
    inputs:                            # what does it require to operate?
      - name: string
        type: string                   # object_type or primitive
        required: boolean
        description: string
    outputs:                           # what does it produce?
      - name: string
        type: string
        guaranteed: boolean
        description: string
    constraints:                       # hard limits that must always hold
      - id: string                     # e.g. CON-001
        statement: string
        enforcement: COMPILE | RUNTIME | REVIEW
    assumptions:                       # what must be true for this object to work?
      - id: string                     # e.g. ASM-001
        statement: string
        validated_by: string           # evidence ID or "assumed"
    guarantees:                        # what this object promises if constraints hold
      - id: string                     # e.g. GUA-001
        statement: string
        condition: string              # "if {assumption} then {guarantee}"
    dependencies:                      # objects this object relies on
      - knowledge_id: string
        reason: string
        strength: HARD | SOFT | OPTIONAL
    risks:                             # known risks
      - id: string                     # e.g. RSK-001
        description: string
        severity: CRITICAL | HIGH | MEDIUM | LOW
        mitigation: string
    examples:                          # concrete usage examples
      - title: string
        context: string
        result: string
    anti_patterns:                     # what NOT to do
      - id: string                     # e.g. AP-001
        name: string
        description: string
        instead: string
```

---

## Canonical Example

```yaml
# For: KNW-PLT-MOD-001 (Quota Manager)
intelligence:
  self_describing:
    purpose: >
      Enforce per-tenant resource quotas for the platform, preventing
      any single tenant from consuming more than their allocated share.

    responsibilities:
      - "Maintain quota state per tenant"
      - "Reject requests that exceed quota limits"
      - "Expose current quota consumption via Registry"
      - "Emit QUOTA_EXCEEDED events on violation"

    inputs:
      - name: tenant_id
        type: string
        required: true
        description: "Unique identifier for the tenant"
      - name: resource_type
        type: ENUM(CPU, MEMORY, REQUESTS, STORAGE)
        required: true
        description: "The resource category to check"
      - name: requested_amount
        type: float
        required: true
        description: "Amount of resource being requested"

    outputs:
      - name: quota_decision
        type: ENUM(ALLOW, DENY)
        guaranteed: true
        description: "Whether the request is within quota"
      - name: remaining_quota
        type: float
        guaranteed: true
        description: "Remaining quota after this decision"

    constraints:
      - id: CON-001
        statement: "Quota check must complete in < 5ms P99"
        enforcement: RUNTIME
      - id: CON-002
        statement: "Quota state must be persisted before returning ALLOW"
        enforcement: RUNTIME
      - id: CON-003
        statement: "Quota limits are read-only from this module; writes go through Admin API"
        enforcement: COMPILE

    assumptions:
      - id: ASM-001
        statement: "Quota configuration is loaded at startup and cached"
        validated_by: "KNW-TEST-TST-001"
      - id: ASM-002
        statement: "Registry is available when quota check runs"
        validated_by: "KNW-TEST-BENCH-001"

    guarantees:
      - id: GUA-001
        statement: "No tenant exceeds quota under concurrent load"
        condition: "if Registry is available and quota config is loaded"
      - id: GUA-002
        statement: "QUOTA_EXCEEDED event is emitted within 10ms of denial"
        condition: "if event bus is operational"

    dependencies:
      - knowledge_id: KNW-PLT-REQ-001
        reason: "Implements this requirement"
        strength: HARD
      - knowledge_id: KNW-RT-RT-001
        reason: "Uses runtime registry for quota state"
        strength: HARD

    risks:
      - id: RSK-001
        description: "Quota state cache stale under split-brain scenario"
        severity: HIGH
        mitigation: "Registry uses consensus protocol (see KNW-RT-RT-001)"
      - id: RSK-002
        description: "Quota check adds latency to every request path"
        severity: MEDIUM
        mitigation: "In-process cache reduces to < 1ms P50"

    examples:
      - title: "Tenant under quota"
        context: "tenant_id=acme, resource=CPU, requested=0.5, limit=4.0, used=2.1"
        result: "ALLOW, remaining=1.4"
      - title: "Tenant at limit"
        context: "tenant_id=beta, resource=REQUESTS, requested=1, limit=1000, used=1000"
        result: "DENY, remaining=0"

    anti_patterns:
      - id: AP-001
        name: "Bypassing quota for internal services"
        description: "Calling platform APIs without going through quota check"
        instead: "Use the QuotaManager for all resource requests including internal ones"
      - id: AP-002
        name: "Synchronous quota reset"
        description: "Resetting quota mid-request instead of at period boundary"
        instead: "Use the scheduled quota reset workflow (WF-002)"
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KIL-001 | Every CANONICAL object MUST have `intelligence.self_describing.purpose` |
| KIL-002 | Every CANONICAL object MUST have ≥ 1 `responsibilities` entry |
| KIL-003 | Every `constraint` MUST have an `enforcement` level |
| KIL-004 | Every `guarantee` MUST reference at least one `assumption` via its condition |
| KIL-005 | Every `risk` MUST have a `mitigation` — "accepted" is a valid value |
| KIL-006 | Every `anti_pattern` MUST have an `instead` field — never leave it empty |
| KIL-007 | `inputs` marked `required: true` may not be undefined at runtime |
| KIL-008 | `outputs` marked `guaranteed: true` must be produced in all non-error paths |

---

## Anti-Patterns

| Pattern | Problem | Fix |
|---------|---------|-----|
| Purpose = object name | Circular — adds no information | Explain WHY, not WHAT |
| Empty responsibilities | Object has no stated contract | List at least one owned behavior |
| Risk without mitigation | Risk is unaddressed | Add mitigation or explicitly accept |
| Assumption without validation | Unverified assumption | Cite test or evidence |

---

## Cross-References

- Executable rules → `02-EXECUTABLE-KNOWLEDGE`
- Semantic capabilities → `03-SEMANTIC-KNOWLEDGE`
- Cortex (why/why-not) → `19-KNOWLEDGE-CORTEX`
- Evidence types → Phase 3.0C.5 `13-CANONICAL-EVIDENCE`
