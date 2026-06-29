---
knowledge_id: KNW-KOS-ARCH-003
title: "KOS Golden Rules"
status: AUTHORITATIVE
phase: "3.0A"
version: "1.0.0"
created: "2026-06-30"
purpose: "Define the 10 inviolable rules that govern the entire Knowledge Operating System"
canonical_source: "architecture/docs/kos/03-GOLDEN-RULES.md"
dependencies:
  - "01-VISION.md"
  - "02-CORE-PHILOSOPHY.md"
related_documents:
  - "11-GOVERNANCE.md"
  - "17-ARCHITECTURE-FREEZE.md"
acceptance_criteria:
  - "All 10 rules are stated"
  - "Each rule has a machine-verifiable enforcement mechanism"
  - "Rules are numbered GR-001 through GR-010"
verification_checklist:
  - "[ ] All 10 rules present"
  - "[ ] Each rule has a verification method"
  - "[ ] No rule contradicts another"
future_extensions:
  - "Additional rules as KOS evolves beyond Phase 3.0A"
---

# KOS Golden Rules

These rules are inviolable. No exception may be granted without a new Architecture Decision
Record and unanimous Platform Team approval.

---

## GR-001 — Knowledge is the Only Canonical Source

**Rule:** No document, database, comment, or codebase may be considered authoritative
unless it is a Knowledge Object registered in the Knowledge Registry.

**Enforcement:**
```bash
platform-kos validate --canonical
# Exits non-zero if any artifact lacks a registered Knowledge Object
```

**Violation:** Any artifact (API, module, test, deployment) without a Knowledge Object
is considered unverified and may not be deployed.

---

## GR-002 — Everything is a Knowledge Object

**Rule:** Every entity in the AI Studio ecosystem — platform component, runtime, service,
API endpoint, algorithm, product, agent, prompt, decision, test, deployment — must be
represented as a typed Knowledge Object in the KOS Registry.

**Enforcement:**
```bash
platform-kos validate --objects
# Checks that all registered entities have Knowledge Object representations
```

**Violation:** An entity without a Knowledge Object cannot be registered in any runtime
or used by any service.

---

## GR-003 — No Runtime Without Knowledge

**Rule:** No runtime may start unless all its modules, capabilities, and services have
corresponding Knowledge Objects in CANONICAL or APPROVED state.

**Enforcement:**
```python
# In PlatformRuntime.initialize()
kos_client.assert_runtime_knowledge_complete(self.runtime_id)
# Raises KOSViolationError if any module lacks a Knowledge Object
```

**Violation:** Runtime boot is aborted. Error: `KOSViolationError: runtime knowledge incomplete`.

---

## GR-004 — No API Without Knowledge

**Rule:** Every HTTP endpoint must have a Knowledge Object of type `APIEndpoint`
containing its contract, inputs, outputs, and the knowledge object it serves.

**Enforcement:**
```bash
platform-kos validate --apis
# Compares registered API routes against APIEndpoint knowledge objects
```

**Violation:** Undocumented endpoints are blocked by the API gateway.

---

## GR-005 — No Implementation Without Knowledge

**Rule:** No code may be merged into `platform/` unless it is linked to a Knowledge Object
of type `Module`, `Service`, `Algorithm`, or `Pattern` in the Knowledge Registry.

**Enforcement:**
```yaml
# Pre-merge CI check
- name: KOS link check
  run: platform-kos validate --implementation --path platform/
```

**Violation:** PR is blocked. CI exits non-zero.

---

## GR-006 — No Test Without Knowledge

**Rule:** Every test must reference the Knowledge Object it is testing via a
`knowledge_id` annotation. Tests without knowledge references are treated as
unverified and excluded from coverage metrics.

**Enforcement:**
```python
# Every test must have:
@pytest.mark.kos("KNW-KOS-ARCH-006")
def test_compiler_generates_python():
    ...
```

**Violation:** `platform-kos coverage` reports the test as unlinked. Unlinked tests
do not count toward knowledge coverage requirements.

---

## GR-007 — No Documentation Outside Knowledge

**Rule:** All documentation (README, architecture docs, API docs, user guides) must be
generated from Knowledge Objects or registered as Knowledge Objects themselves.
Manually written documentation that is not linked to KOS is invalid.

**Enforcement:**
```bash
platform-kos validate --docs
# Checks all .md files in architecture/ and docs/ for knowledge_id front-matter
```

**Violation:** Undocumented or orphaned documentation is quarantined (moved to `archive/`
on next cleanup run).

---

## GR-008 — Everything Must Be Traceable

**Rule:** Every artifact must have a complete traceability path:
```
Requirement → Architecture Decision → Specification → Implementation → Test → Deployment
```
No step may be skipped.

**Enforcement:**
```bash
platform-kos trace --artifact platform/runtimes/ai_runtime/
# Outputs full traceability chain or exits non-zero if any step is missing
```

**Violation:** Missing traceability step blocks deployment.

---

## GR-009 — Everything Must Be Versioned

**Rule:** Every Knowledge Object must have a version field following semantic versioning.
Version must increment when the object's content changes materially.
Version history must be preserved (not overwritten).

**Enforcement:**
```bash
platform-kos validate --versions
# Checks all objects have valid semver and version history
```

**Violation:** Object with missing or unchanged version on content update is rejected
by the Knowledge Registry.

---

## GR-010 — Every Decision Must Have Evidence

**Rule:** Every Architecture Decision Record (ADR) and every Knowledge Object with
`decision_type` must have an `evidence` field containing:
- The context that required the decision
- The alternatives considered
- The rationale for the chosen option
- The expected consequences

**Enforcement:**
```bash
platform-kos validate --evidence
# Checks all decision-type objects for non-empty evidence field
```

**Violation:** Decision without evidence is demoted to DRAFT status regardless of
approval state.

---

## Rules Summary Table

| Rule | Name | Verification Command |
|------|------|---------------------|
| GR-001 | Knowledge is canonical | `platform-kos validate --canonical` |
| GR-002 | Everything is a Knowledge Object | `platform-kos validate --objects` |
| GR-003 | No runtime without knowledge | Runtime boot assertion |
| GR-004 | No API without knowledge | `platform-kos validate --apis` |
| GR-005 | No implementation without knowledge | CI pre-merge check |
| GR-006 | No test without knowledge | `platform-kos coverage` |
| GR-007 | No docs outside knowledge | `platform-kos validate --docs` |
| GR-008 | Everything traceable | `platform-kos trace` |
| GR-009 | Everything versioned | `platform-kos validate --versions` |
| GR-010 | Every decision has evidence | `platform-kos validate --evidence` |

All 10 checks combined:
```bash
platform-kos validate --all
# Runs all 10 checks; exits 0 only if all pass
```
