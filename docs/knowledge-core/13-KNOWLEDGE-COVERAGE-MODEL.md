---
knowledge_id: KA-SPEC-013
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.0.5
created: 2026-06-29
review_date: 2026-12-29
canonical: true
domain: DOM-KNOWLEDGE
capability: knowledge-core
type: specification
implements:
  - KA-SPEC-001
depends_on:
  - id: KA-SPEC-012
    reason: "Coverage derives from evidence"
  - id: KA-SPEC-011
    reason: "Coverage is the output of the traceability chain"
---

# Knowledge Coverage Model

## Measuring How Completely Architecture Is Designed, Built, and Tested

---

## 1. Purpose

Coverage answers: **"How much of what should exist, actually exists?"**

The Coverage Model defines four coverage dimensions, how they are measured at document/capability/domain levels, how gaps are detected, and how coverage data is declared in knowledge documents.

---

## 2. Coverage Dimensions

| Dimension | Question It Answers | Data Source |
|-----------|--------------------|-----------  |
| **Architecture Coverage** | What % of the capability design is documented? | Declared by owner in `coverage.architecture` |
| **Implementation Coverage** | What % of the architecture is implemented? | Derived from `implemented_by` evidence with `verified: true` |
| **Test Coverage** | What % of the implementation is tested? | Derived from `tested_by` evidence with `coverage_percent` |
| **Desktop Coverage** | What % of the desktop UI is documented? | Declared by owner in `coverage.desktop` |

---

## 3. Coverage Declaration

### 3.1 In Document Frontmatter

```yaml
coverage:
  architecture: 90     # 0-100: % of the capability's design documented
  implementation: 75   # 0-100: % of architecture with verified implementation
  testing: 60          # 0-100: % of implementation with passing tests
  desktop: 0           # 0-100: % of UI documented (0 = not applicable)
```

### 3.2 Ownership of Coverage Values

| Dimension | Who Sets It | How |
|-----------|------------|-----|
| `architecture` | Document owner | Self-assessed; reviewed at review_date |
| `implementation` | Index engine | Computed from `implemented_by` evidence (verified=true) |
| `testing` | Index engine | Computed from `tested_by` evidence (coverage_percent) |
| `desktop` | Document owner | Self-assessed |

**Rule:** `implementation` and `testing` values declared manually in frontmatter are **overridden** by the index engine's computed values when evidence is available. This prevents false claims.

---

## 4. Coverage Computation Algorithm

### 4.1 Implementation Coverage (Computed)

```
FUNCTION compute_implementation_coverage(obj: KnowledgeObject) → Integer:

  IF obj.implemented_by IS EMPTY:
    RETURN 0

  verified_refs = [r for r in obj.implemented_by WHERE r.verified = true]
  total_refs = count(obj.implemented_by)

  IF total_refs = 0: RETURN 0

  base_coverage = (count(verified_refs) / total_refs) × 100

  -- Confidence discount:
  high_confidence = [r for r in verified_refs WHERE r.confidence = 'high']
  confidence_factor = count(high_confidence) / count(verified_refs)

  RETURN ROUND(base_coverage × (0.5 + 0.5 × confidence_factor))
```

### 4.2 Test Coverage (Computed)

```
FUNCTION compute_test_coverage(obj: KnowledgeObject) → Integer:

  IF obj.tested_by IS EMPTY:
    -- Fall back to testing.md declared value if available
    testing_doc = get_sibling(obj, type=testing)
    IF testing_doc.coverage.testing > 0:
      RETURN testing_doc.coverage.testing
    RETURN 0

  verified_tests = [t for t in obj.tested_by WHERE t.verified = true]

  IF count(verified_tests) = 0: RETURN 0

  -- Use coverage_percent from test evidence if available
  IF ALL(t.coverage_percent > 0 for t in verified_tests):
    RETURN ROUND(AVG(t.coverage_percent for t in verified_tests))

  -- Fall back to binary: has tests = 40%
  RETURN 40
```

### 4.3 Capability Coverage Aggregation

```
FUNCTION compute_capability_coverage(capability):
  docs = capability.get_all_documents(status=approved)
  arch = docs[type=architecture]

  RETURN CoverageMap {
    architecture:    arch.coverage.architecture IF arch ELSE 0
    implementation:  compute_implementation_coverage(arch) IF arch ELSE 0
    testing:         compute_test_coverage(arch) IF arch ELSE 0
    desktop:         docs[type=desktop].coverage.desktop IF exists ELSE 0
  }
```

---

## 5. Coverage Thresholds

### 5.1 Document-Level Thresholds

| Dimension | Threshold | Status |
|-----------|----------|--------|
| Architecture | < 50 | Poor |
| Architecture | 50–79 | Fair |
| Architecture | ≥ 80 | Good |
| Implementation | < 40 | Poor |
| Implementation | 40–69 | Fair |
| Implementation | ≥ 70 | Good |
| Testing | < 30 | Poor |
| Testing | 30–59 | Fair |
| Testing | ≥ 60 | Good |
| Desktop | < 60 | Poor (if applicable) |

### 5.2 Capability-Level Phase Gate Thresholds

These coverage levels are required before a capability is considered "production ready" in Phase 2.0D+:

| Dimension | Required for Phase Gate |
|-----------|------------------------|
| Architecture | ≥ 80 |
| Implementation | ≥ 70 |
| Testing | ≥ 60 |
| Desktop | ≥ 70 (if capability has UI) |

---

## 6. Gap Detection

### 6.1 Architecture Gaps

Architecture gaps are detected by comparing declared capability design to existing documentation:

```
FUNCTION find_architecture_gaps(capability):
  gaps = []

  expected_sections = [
    "Design Goals", "Core Design", "Component Model",
    "Data Flow", "Key Decisions", "Constraints"
  ]

  arch_doc = capability.get(architecture)
  IF arch_doc IS NULL:
    gaps.append(GAP("No architecture document exists", severity=critical))
    RETURN gaps

  present_headings = extract_h2_headings(arch_doc.body)

  FOR each section in expected_sections:
    IF section not in present_headings:
      gaps.append(GAP("Missing section: " + section, severity=high))

  RETURN gaps
```

### 6.2 Implementation Gaps

```
FUNCTION find_implementation_gaps(capability):
  arch = capability.get(architecture)
  IF arch IS NULL: RETURN [GAP("No architecture")]

  unimplemented = [
    section for section in arch.get_implementation_sections()
    WHERE no evidence.implemented_by references section
  ]

  RETURN [GAP("Section not implemented: " + s) for s in unimplemented]
```

### 6.3 Test Gaps

```
FUNCTION find_test_gaps(capability):
  gaps = []

  IF capability.testing_doc IS NULL:
    gaps.append(GAP("No testing.md document", severity=high))
    RETURN gaps

  IF compute_test_coverage(capability) < 60:
    gaps.append(GAP(
      "Test coverage " + coverage + "% is below 60% threshold",
      severity=high
    ))

  RETURN gaps
```

---

## 7. Coverage Report

The KIE generates a Coverage Report as part of the health report:

```markdown
## Coverage Report (generated 2026-06-29)

### Repository Coverage Summary

| Dimension | Coverage | Status |
|-----------|---------|--------|
| Architecture | 74% | Fair |
| Implementation | 58% | Poor |
| Testing | 42% | Poor |

### By Capability

| Capability | Architecture | Implementation | Testing | Phase Gate |
|-----------|-------------|----------------|---------|-----------|
| workflow-runtime | 90% ✓ | 75% ✓ | 60% ✓ | PASS |
| prompt-os | 70% △ | 40% △ | 30% ✗ | FAIL |
| provider-framework | 80% ✓ | 65% △ | 50% △ | FAIL |
| conversation-intelligence | 60% △ | 30% ✗ | 20% ✗ | FAIL |

### Critical Gaps

1. [CRIT] conversation-intelligence: No implementation evidence
2. [HIGH] prompt-os: Testing coverage 30% (target: 60%)
3. [HIGH] provider-framework: Implementation coverage 65% (target: 70%)
```

---

## 8. Coverage in the Knowledge Graph

Coverage creates weighted edges:

```
KA-ARCH-001 ──[has_coverage]──▶ CoverageNode {
  architecture: 90,
  implementation: 75,
  testing: 60,
  computed_at: "2026-06-29"
}
```

Coverage nodes are computed artifacts — they are never authored manually.

---

## References

- [12-KNOWLEDGE-EVIDENCE-MODEL.md](12-KNOWLEDGE-EVIDENCE-MODEL.md) — Evidence drives implementation and test coverage
- [11-KNOWLEDGE-TRACEABILITY-MODEL.md](11-KNOWLEDGE-TRACEABILITY-MODEL.md) — Coverage is the measurable output of traceability
- [KA-STD-006](../knowledge-architecture/KNOWLEDGE-HEALTH-METRICS.md) — Health metrics include coverage scores
