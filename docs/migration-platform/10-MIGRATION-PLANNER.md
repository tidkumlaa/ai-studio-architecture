---
knowledge_id: KA-PLAT-011
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.1
created: 2026-06-29
review_date: 2026-12-29
canonical: true
domain: DOM-KNOWLEDGE
capability: migration-platform
type: specification
depends_on:
  - id: KA-PLAT-010
    reason: "Consumes HealthReport to determine migration scope"
  - id: KA-ROAD-001
    reason: "6-phase migration plan this component operationalizes"
---

# Migration Planner

## Execution Order, Risk Analysis, and Rollback Planning

---

## 1. Purpose

The MigrationPlanner synthesizes all scanner and analyzer outputs into an
actionable `MigrationPlan`. The plan specifies what to do, in what order, with
what risk, and how to roll back if anything goes wrong.

---

## 2. Migration Phases

The planner generates six phases, mirroring KA-ROAD-001:

| Phase | Name | Key Actions | Est. Duration |
|-------|------|-------------|--------------|
| A | Foundation | Create directory structure, git branch, checkpoints | 0.5d |
| B | Classification | Classify all documents, generate initial inventory | 1-2d |
| C | Archive | Move dead/obsolete documents to archive/ | 0.5-1d |
| D | Split | Split all oversized documents | 1-5d |
| E | Metadata | Generate and apply frontmatter to all documents | 2-7d |
| F | Index | Generate all indexes, run audit, health gate check | 0.5-1d |

Duration estimates scale with `documents_without_metadata` and `oversized_count`.

---

## 3. Execution Order

Within each phase, documents are processed in dependency order:

```
COMPUTE_EXECUTION_ORDER(metadata_map, relationships):
  # Topological sort of dependency graph
  graph = build_dependency_graph(metadata_map, relationships)
  order = topological_sort(graph)  # Kahn's algorithm
  
  # Documents with no dependencies first
  # Documents that others depend on before their dependents
  RETURN order
```

Cycles are broken by choosing the document with fewer inbound edges first and
logging a WARNING.

---

## 4. Risk Analysis

```yaml
RiskReport:
  overall_risk: HIGH  # LOW | MEDIUM | HIGH | CRITICAL
  items:
    - description: "AI-STUDIO-MASTER-ARCHITECTURE.md has 8,008 lines"
      risk_level: CRITICAL
      mitigation: "Split into capability documents before metadata generation"
      affected_count: 1
      
    - description: "23 documents exceed 500 lines"
      risk_level: HIGH
      mitigation: "Run splitter before metadata generation"
      affected_count: 23
      
    - description: "786 documents have no frontmatter"
      risk_level: HIGH
      mitigation: "Metadata generation is the primary Phase E task"
      affected_count: 786
      
    - description: "47 broken links detected"
      risk_level: MEDIUM
      mitigation: "Fix after split phase; paths will change during splitting"
      affected_count: 47
```

---

## 5. Risk Level Determination

| Condition | Risk Level |
|-----------|-----------|
| Any document > 5,000 lines | CRITICAL |
| > 10% documents oversized | HIGH |
| > 50% documents without metadata | HIGH |
| Broken link ratio > 10% | MEDIUM |
| Orphan ratio > 20% | MEDIUM |
| Health score > 60 | LOW |

Overall risk = maximum of all individual risk levels.

---

## 6. MigrationPlan Structure

```yaml
MigrationPlan:
  repository_path: str
  plan_generated: "2026-06-29"
  target_health_score: 80          # Phase gate requirement
  current_health_score: 15.2
  
  summary:
    total_documents: 786
    total_oversized: 23
    without_metadata: 774
    broken_links: 47
    orphans: 31
    estimated_total_days: 12
    
  phases:
    - phase: "A"
      name: "Foundation"
      estimated_days: 0.5
      actions:
        - "Create git branch: knowledge-arch-refactoring"
        - "Create checkpoint tag: phase-A-start"
      risk_level: LOW
      
    - phase: "B"
      name: "Classification"
      estimated_days: 1.0
      actions:
        - "Run classifier on all 786 documents"
        - "Generate classification-report.yaml"
      risk_level: LOW
      
    - phase: "C"
      name: "Archive"
      estimated_days: 0.5
      actions:
        - "Move 5 dead documents to archive/"
        - "Move 3 superseded documents to archive/"
      documents: ["docs/old/...", ...]
      risk_level: MEDIUM
      
    - phase: "D"
      name: "Split"
      estimated_days: 2.0
      actions:
        - "Split AI-STUDIO-MASTER-ARCHITECTURE.md (8008 lines)"
        - "Split 22 additional oversized documents"
      documents: ["AI-STUDIO-MASTER-ARCHITECTURE.md", ...]
      risk_level: HIGH
      
    - phase: "E"
      name: "Metadata"
      estimated_days: 7.0
      actions:
        - "Generate frontmatter for 774 documents"
        - "Repair 12 documents with partial metadata"
      risk_level: MEDIUM
      
    - phase: "F"
      name: "Index Generation"
      estimated_days: 1.0
      actions:
        - "Generate KNOWLEDGE-REGISTRY.yaml"
        - "Generate all capability index.yaml files"
        - "Generate KNOWLEDGE-INDEX.md"
        - "Run health check: target ≥ 80/100"
      risk_level: LOW
      
  rollback_plan:
    strategy: "git"
    branch: "knowledge-arch-refactoring"
    phase_tags:
      - "phase-A-complete"
      - "phase-B-complete"
      - "phase-C-complete"
      - "phase-D-complete"
      - "phase-E-complete"
      - "phase-F-complete"
    estimated_rollback_time_minutes: 5
```

---

## 7. Phase Gate

After Phase F, the planner verifies the health gate:

```
VERIFY_PHASE_GATE(health_report):
  IF health_report.repository_score >= 80:
    MARK phase_gate = PASS
    ENABLE next phase: 2.0D.2
  ELSE:
    MARK phase_gate = FAIL
    REPORT gap = 80 - health_report.repository_score
    GENERATE remediation_plan
```

---

## References

- [KA-ROAD-001](../knowledge-architecture/REPOSITORY-REFACTORING-PLAN.md) — The 6-phase plan
- [09-HEALTH-ANALYZER.md](09-HEALTH-ANALYZER.md) — HealthReport source
- [KA-VIS-001](../knowledge-architecture/KNOWLEDGE-ARCHITECTURE-VISION.md) — Phase gate criteria
