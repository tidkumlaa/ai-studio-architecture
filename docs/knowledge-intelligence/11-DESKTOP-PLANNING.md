---
knowledge_id: KA-KIP-012
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.2
created: 2026-06-29
review_date: 2026-12-29
canonical: true
domain: DOM-KNOWLEDGE
capability: knowledge-intelligence
type: specification
---

# Desktop Planning

## 9 Dashboard Designs — Architecture Only

> **Constraint:** Design only. No implementation. All dashboards are Phase 2.0D.3+.
> Backend data sources from the Knowledge Intelligence Platform are fully specified.

---

## 1. Knowledge Explorer

**Purpose:** Browse and navigate all knowledge objects by domain, capability, type, status.

**Data source:** KnowledgeRegistry, KnowledgeCatalog, KnowledgeSearch

```
┌──────────────────────┬────────────────────────────────────────────────┐
│ Navigation           │ workflow-runtime capability (23 docs)          │
│                      │                                                │
│ ▼ DOM-WORKFLOW       │ [Search: ___________] [Type ▾] [Status ▾]     │
│   ▶ workflow-runtime │                                                │
│   ▶ event-bus        │ ● KA-ARCH-001  architecture  approved  72/100 │
│   ▶ scheduler        │ ○ KA-SPEC-005  specification  draft    45/100 │
│                      │ ○ KA-DB-001    database       draft    38/100 │
│ ▶ DOM-AI             │ ✗ testing.md   testing        missing         │
│ ▶ DOM-KNOWLEDGE      │                                                │
│ ▶ DOM-PROVIDER       │ [Open] [Query] [Compile PromptPack]           │
└──────────────────────┴────────────────────────────────────────────────┘
```

---

## 2. Graph Viewer

**Purpose:** Interactive visualization of the knowledge graph.

**Data source:** KnowledgeGraph, graph.json

```
┌────────────────────────────────────────────────────────────────────┐
│  [Layout ▾] [Filter ▾] [Edge Types ▾] [Depth: 3 ▾]               │
│                                                                    │
│     ○ KA-SPEC-001 ──────────────────▶ ● KA-ARCH-001              │
│        implements │                        │ depends_on            │
│                   │                        ▼                       │
│              ○ KA-SPEC-008         ○ KA-SPEC-020                  │
│                                                                    │
│  Selected: KA-ARCH-001 "Workflow Runtime Architecture"             │
│  Health: 72/100 | In: 3 | Out: 5 | Cycle: No                     │
│                                                                    │
│  [Find Path] [Show Impact] [Show Orphans] [Export JSON]           │
└────────────────────────────────────────────────────────────────────┘
```

---

## 3. Impact Explorer

**Purpose:** Show what is affected when a document changes.

**Data source:** ImpactAnalyzer

```
┌────────────────────────────────────────────────────────────────────┐
│  Impact Analysis: KA-SPEC-001                                      │
│  Changed: "Knowledge Object Specification"                         │
│                                                                    │
│  ┌─────────────────────────────────┐  ┌──────────────────────┐   │
│  │ Direct Impact (4)               │  │ Severity: HIGH       │   │
│  │  ● KA-ARCH-001  architecture   │  │                      │   │
│  │  ○ KA-SPEC-020  specification  │  │ Arch affected: 2     │   │
│  │  ○ KA-SPEC-009  specification  │  │ Tests affected: 7    │   │
│  │  ○ KA-DB-001    database       │  │ Docs affected: 18    │   │
│  └─────────────────────────────────┘  └──────────────────────┘   │
│                                                                    │
│  Transitive Impact (14 more)...   [Expand] [Export] [KQL Query]  │
└────────────────────────────────────────────────────────────────────┘
```

---

## 4. Coverage Explorer

**Purpose:** Architecture/implementation/testing/desktop coverage per capability.

**Data source:** HealthRuntime, CoverageAnalyzer

```
┌────────────────────────────────────────────────────────────────────┐
│  Coverage Dashboard                                                │
│                                                                    │
│  Capability              Arch   Impl   Test   Desktop   Overall   │
│  ─────────────────────────────────────────────────────────────── │
│  workflow-runtime         ●90%   ○60%   ○40%   ✗0%       ○65%    │
│  knowledge-core           ●95%   ✗0%    ✗0%    ✗0%       ○28%    │
│  provider-framework       ●80%   ○70%   ○50%   ✗0%       ○60%    │
│                                                                    │
│  ● ≥80%  ○ 40-79%  ✗ <40%                                        │
│                                                                    │
│  [Filter] [Export] [View Gaps] [Fix Priority]                     │
└────────────────────────────────────────────────────────────────────┘
```

---

## 5. Evidence Viewer

**Purpose:** Inspect evidence records for any knowledge object.

**Data source:** EvidenceEngine

```
┌────────────────────────────────────────────────────────────────────┐
│  Evidence: KA-ARCH-001 "Workflow Runtime Architecture"             │
│                                                                    │
│  Overall Confidence: 0.78                                          │
│                                                                    │
│  ✓ [EV-001] source_file  verified   0.90  tools/orchestrator/...  │
│  ✓ [EV-002] test         validated  0.75  tests/test_workflow.py  │
│  ○ [EV-003] schema       collected  0.50  schemas/workflow.yaml   │
│  ✗ [EV-004] log          expired    0.00  ci/run-2026-01-15       │
│                                                                    │
│  [Add Evidence] [Re-verify] [View History]                        │
└────────────────────────────────────────────────────────────────────┘
```

---

## 6. Reasoning Viewer

**Purpose:** Show inferred relationships and reasoning chains.

**Data source:** KnowledgeReasoning

```
┌────────────────────────────────────────────────────────────────────┐
│  Reasoning: KA-ARCH-001                                            │
│                                                                    │
│  Inferred Relationships (3):                                       │
│  • IR-001: implemented_by KA-IMPL-003  conf=0.95  [declared impl]│
│  • IR-002: transitively_depends_on KA-SPEC-004  conf=0.80  [chain]│
│  • IR-007: has_coverage_gap  conf=0.90  [no testing evidence]     │
│                                                                    │
│  Transitive Dependencies (8):                                      │
│    KA-SPEC-001 → KA-SPEC-008 → KA-SPEC-016 → ...                 │
│                                                                    │
│  [Show Rule Details] [Export Inference Chain]                     │
└────────────────────────────────────────────────────────────────────┘
```

---

## 7. Search Explorer

**Purpose:** Full-featured search with hybrid ranking and filters.

**Data source:** KnowledgeSearch

```
┌────────────────────────────────────────────────────────────────────┐
│  Search: [workflow execution engine_______] [Search]               │
│  Filters: [Domain ▾] [Type ▾] [Status ▾] [Health ▾]              │
│                                                                    │
│  23 results (12ms)                                                 │
│                                                                    │
│  ● KA-ARCH-001  0.94  "Workflow Runtime Architecture"             │
│    workflow-runtime · approved · 72/100                           │
│    "...the execution engine orchestrates..."                       │
│                                                                    │
│  ○ KA-SPEC-005  0.87  "Workflow Runtime Specification"            │
│    workflow-runtime · draft · 45/100                              │
│    "...defines the execution contract for..."                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## 8. KQL Console

**Purpose:** Interactive KQL query interface with syntax highlighting.

**Data source:** KQLEngine

```
┌────────────────────────────────────────────────────────────────────┐
│  KQL Console                              [Run ▶] [Save] [History]│
│                                                                    │
│  SELECT id, title, capability, health_score                       │
│  FROM knowledge                                                    │
│  WHERE domain = "DOM-WORKFLOW"                                     │
│    AND status = "approved"                                         │
│  ORDER BY health_score DESC                                        │
│  LIMIT 10                                                          │
│                                                                    │
│  ─────────────────────────────────────────────────────────────── │
│  Results (7 rows, 4ms, index used: catalog.by_domain)             │
│                                                                    │
│  id           title                     capability   health       │
│  KA-ARCH-001  Workflow Runtime Arch...  workflow-rt  72.0         │
└────────────────────────────────────────────────────────────────────┘
```

---

## 9. Health Dashboard

**Purpose:** Repository health overview with trends and remediation priorities.

**Data source:** HealthRuntime, KnowledgeDebt

```
┌────────────────────────────────────────────────────────────────────┐
│  Repository Health  Score: 44.5/100  [Phase Gate: FAIL ✗]         │
│                                                                    │
│  Structural  ████████░░  32%    Evidence  ░░░░░░░░░░   0%        │
│  Freshness   ██████████  95%    Coverage  ████░░░░░░  40%        │
│  Relations   ████████░░  80%    Consist.  ████████░░  80%        │
│                                                                    │
│  Knowledge Debt: 38.4 days                                        │
│                                                                    │
│  TOP PRIORITIES              Impact    Effort                     │
│  1. Add metadata to 92 docs  +18pts    7 days                    │
│  2. Split 29 oversized docs  +8pts     2 days                    │
│  3. Fix 47 broken links      +6pts     0.5 days                  │
│                                                                    │
│  [Generate Report] [View Worst] [Start Migration]                 │
└────────────────────────────────────────────────────────────────────┘
```

---

## References

- All components in this phase provide backend APIs for these dashboards
- Phase 2.0D.3 will implement the desktop panels using these designs
