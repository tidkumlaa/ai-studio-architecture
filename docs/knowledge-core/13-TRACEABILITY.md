# KNW-KC-ARCH-013 — Traceability

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Traceability establishes the complete, unbroken chain from a Requirement through Architecture, Decision, Specification, Module, Service, API, Tests, Deployment, and Monitoring. Every link must be explicit, typed, and machine-traversable.

---

## Canonical Traceability Chain

```
Requirement
    ↓  [implements]
Architecture
    ↓  [documents]
Decision
    ↓  [implements]
Specification
    ↓  [implements]
Module
    ↓  [uses]
Service
    ↓  [uses]
API
    ↓  [tests]
Test
    ↓  [verifies → passes]
Deployment
    ↓  [monitors]
Monitoring
    ↓  [generates → feeds back to]
Requirement (loop closed)
```

---

## Traceability Record

Each step in the chain is tracked by a `TraceabilityRecord`:

```yaml
traceability_record:
  trace_id: "TRACE-{source_id}-{target_id}"
  source_id: "KNW-KOS-REQ-001"
  target_id: "KNW-KOS-ARCH-001"
  trace_type: FORWARD|REVERSE|IMPACT|DEPENDENCY
  relation_type: implements|documents|tests|verifies|...
  confidence: float
  coverage_contribution: float      # what fraction of source does target cover?
  gap_detected: bool
  verified_at: datetime | null
  verified_by: string | null
```

---

## Traceability Traversals

### Forward Traversal
Answers: "What does requirement REQ-001 trace to in production?"

```
FORWARD_TRACE(requirement_id):
  1. requirement → [implements] → architecture_ids
  2. architecture → [documents] → decision_ids
  3. architecture → [implements] → specification_ids
  4. specification → [implements] → module_ids
  5. module → [uses] → service_ids
  6. service → [uses] → api_ids
  7. api → [tests] → test_ids
  8. test → [verifies] → deployment_ids
  Returns: ForwardTraceResult{chain, coverage_score, gaps}
```

### Reverse Traversal
Answers: "What requirements does this deployment satisfy?"

```
REVERSE_TRACE(deployment_id):
  Walk all edges in reverse direction
  Returns: list[RequirementRef] with coverage score
```

### Impact Traversal
Answers: "What will break if I change requirement REQ-001?"

```
IMPACT_TRACE(object_id):
  1. Get all objects that depend_on or implement this object
  2. Recursively expand (DFS, max depth 10)
  3. Classify by impact: CRITICAL|HIGH|MEDIUM|LOW
  4. Returns: ImpactReport{affected_objects, blast_radius, risk_score}
```

Impact severity thresholds (from Phase 3.0A `09-REASONING-ENGINE`):
- CRITICAL: blast_radius ≥ 26 objects
- HIGH: blast_radius 11–25
- MEDIUM: blast_radius 4–10
- LOW: blast_radius 1–3

### Gap Detection
Answers: "Which requirements have no implementation?"

```
GAP_DETECT(namespace):
  For each requirement in namespace:
    If no [implements] edge to any architecture:
      → GAP: UNARCHITECTED
    If architecture exists but no [implements] to module:
      → GAP: UNIMPLEMENTED
    If module exists but no [tests] edge:
      → GAP: UNTESTED
    If test exists but no [verifies] edge to deployment:
      → GAP: UNDEPLOYED
```

---

## Traceability Coverage Score

```
traceability_coverage =
  number_of_requirements_with_full_chain
  ─────────────────────────────────────
  total_requirements_in_scope

full_chain = requirement → architecture → module → test (minimum path)
```

---

## Traceability Rules

### TR-001 — No Orphan Requirements
Every CANONICAL requirement must have at least one Architecture object with an `implements` edge.

### TR-002 — No Untested Modules
Every CANONICAL module with `is_public: true` must have at least one `tests` edge pointing to a TestObject.

### TR-003 — No Undeployed Tests
Every CANONICAL TestObject must have a `verifies` edge to a Deployment or a Module (in-repo testing is acceptable).

### TR-004 — Evidence on Every Link
Every traceability edge must carry `confidence ≥ 0.50`.

### TR-005 — Bidirectional Indexing
Every forward trace is automatically indexed in reverse. Reverse traversal must complete in O(1) lookup, not O(N) scan.

---

## Traceability Matrix Format

```yaml
# traceability-matrix.yaml (auto-generated)
requirements:
  KNW-KOS-REQ-001:
    architecture: [KNW-KOS-ARCH-001, KNW-KOS-ARCH-004]
    decisions: [KNW-KOS-DEC-001]
    modules: [KNW-AI-MOD-001]
    services: [KNW-PLAT-SVC-001]
    tests: [KNW-TEST-UNIT-001, KNW-TEST-INTEG-001]
    deployments: [KNW-DEPLOY-001]
    coverage_score: 0.92
    gaps: []
```

---

## Traceability Engine Protocol

```python
class TraceabilityEngine(Protocol):
    def forward_trace(
        self, requirement_id: str, max_depth: int = 10
    ) -> ForwardTraceResult: ...

    def reverse_trace(
        self, object_id: str
    ) -> list[RequirementRef]: ...

    def impact_analysis(
        self, object_id: str
    ) -> ImpactReport: ...

    def detect_gaps(
        self, namespace: str
    ) -> list[TraceabilityGap]: ...

    def compute_coverage(
        self, namespace: str
    ) -> float: ...

    def generate_matrix(
        self, namespace: str
    ) -> dict: ...

    def verify_chain(
        self, requirement_id: str
    ) -> ChainVerificationResult: ...
```

---

## CLI Integration

```bash
# Forward trace from requirement
platform-kos trace --forward KNW-KOS-REQ-001

# Reverse trace from deployment
platform-kos trace --reverse KNW-DEPLOY-001

# Impact analysis
platform-kos trace --impact KNW-PLAT-SVC-001

# Gap detection for namespace
platform-kos trace --gaps kos

# Generate coverage report
platform-kos trace --coverage kos --output traceability-matrix.yaml
```

---

## Cross-References

- Relationship types used → `07-RELATIONSHIP-TYPES`
- Graph traversal algorithms → `25-GRAPH-ALGORITHMS`
- Gap detection verification → `38-VERIFICATION`
- Coverage in quality score (QD-3) → `11-QUALITY-ENGINE`
