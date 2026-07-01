# ADR-0001 — KIL Phase Classification: Official Spec vs Internal Phase Tracking vs Architecture Extensions

**Status:** Accepted
**Date:** 2026-07-01
**Owner:** architecture-board
**Related:** `architecture/docs/knowledge-intelligence-layer/` (KIL v1.0), `reports/kil/KIL-IMPLEMENTATION-MATRIX.md`, `reports/kil/KIL-DEPENDENCY-GRAPH.md`

---

## Context

A specification-mapping audit (2026-07-01) compared the repository's implementation
work — committed under commit-message labels `feat(phase-4.0B.N)` — against the
frozen Knowledge Intelligence Layer (KIL) architecture in
`architecture/docs/knowledge-intelligence-layer/`.

The audit found that the step number `N` used in commit messages and in
`tools/knowledge/verify.py` docstrings (`"Step 20 — Knowledge Thinking Layer"`,
`"Step 23 — Knowledge Execution Orchestration Layer"`, etc.) does **not** map
1:1 to KIL document number `N`. Three distinct concerns had become conflated
under a single numbering scheme:

1. The **official KIL specification** — 33 frozen documents, each with a fixed
   document number, tier, and `KIL-NNN` rule range.
2. **Internal development phase tracking** — the `Phase 4.0B Step N` label
   used in commit messages, which is a local implementation-sequencing
   convention invented during this work and never defined in any architecture
   document. No `architecture/docs/**` file uses the literal string
   `"Phase 4.0B"`.
3. **Architecture extensions** — functionality implemented and shipped under a
   `Phase 4.0B` step number that has **no backing KIL document at all** (Steps
   21, 23, 24). Two of the three modules (`execution.py`, `stack.py`)
   already self-document this in their own docstrings ("No official Step
   23/24 specification found in architecture/docs").

Without an explicit classification, future sessions risk two failure modes:
(a) assuming "Step N done" implies "KIL Doc N done" (it frequently does not —
e.g. Step 22 implemented KIL Doc 17, not Doc 22), and (b) treating an
architecture extension as if it were a certified part of the frozen KIL v1.0
architecture when the architecture board never reviewed it against a spec,
because none exists.

---

## Decision

We adopt a three-tier classification for all work in this area, going forward:

### Tier A — Official KIL Specification

The sole source of truth is `architecture/docs/knowledge-intelligence-layer/`
(00-README through 30-KNOWLEDGE-FREEZE, plus `README.md` and `index.yaml`).
It is FROZEN as of 2026-06-30 (`index.yaml: status: FROZEN`). Every
`intelligence.<sub_block>` schema and every `KIL-NNN` rule ID is defined
here and nowhere else. Implementation claims must cite the document ID
(`KNW-KIL-DOC-0NN`) and rule range, not a step number.

### Tier B — Internal Development Phase Tracking

`Phase 4.0B Step N` (as used in git commit subjects and some
`verify.py` suite docstrings) is a **local, non-architectural** sequencing
label. It exists only to order the commits in this repository's history and
carries no guarantee that step `N` implements KIL document `N`. The mapping
observed to date:

| Step | What it actually implemented | KIL Doc |
|---|---|---|
| 1–18 | Base Knowledge Intelligence Graph (registry, graph runtime, query, search, reasoning, impact, health, evidence, compiler, context) | Not KIL — implements the earlier, separate **Knowledge Intelligence Platform** spec (`architecture/docs/knowledge-intelligence/`, Phase 2.0D.2, KA-KIP-001..013) |
| 19 | Report generation for the base KIG | Not KIL |
| 20 | Knowledge Thinking Layer | **Doc 20** (match) |
| 21 | "Knowledge Planning Layer" | **No KIL doc** — see Tier C |
| 22 | Knowledge Decision Layer | **Doc 17** (mismatch — not Doc 22, which is Summarization) |
| 23 | "Knowledge Execution Orchestration Layer" | **No KIL doc** — see Tier C |
| 24 | "Knowledge Intelligence Stack Integration & Certification" | **No KIL doc** — see Tier C |
| 25 | Knowledge Intelligence Dashboard | **Doc 25** (match) |
| 26 | Knowledge Intelligence Metrics | **Doc 26** (match) |
| 27 | AI Readiness Score Engine | **Doc 27** (match) |
| 28 | Knowledge Search Optimization | **Doc 28** (match) |
| 29 | Knowledge Canonical Index | **Doc 29** (match) |

Going forward, new work should be labeled by KIL document ID first
(`feat(kil-doc-01): ...`), with any internal step counter kept as a
secondary detail in the commit body, not the sole identifier.

### Tier C — Architecture Extensions

Three modules were implemented under a `Phase 4.0B` step number with **no
corresponding KIL document**:

| Module | Step | Self-declared spec status |
|---|---|---|
| `tools/knowledge/planning.py` | 21 | No disclaimer present in the module docstring (gap — see Consequences) |
| `tools/knowledge/execution.py` | 23 | Docstring: *"No official 'Step 23 / Execution Orchestration' specification was found in architecture/docs"* |
| `tools/knowledge/stack.py` | 24 | Docstring: *"No official 'Step 24...' specification found in architecture/docs"* |

These are legitimate, tested, shipped runtime capabilities. They are **not**
part of the frozen KIL v1.0 architecture and were never reviewed by the
architecture board against a written specification. They must not be counted
toward KIL document coverage in any future audit, dashboard, or certification
claim.

---

## Consequences

1. `reports/kil/KIL-IMPLEMENTATION-MATRIX.md` is the authoritative coverage
   tracker for Tier A work. It supersedes any coverage claim based on
   counting `Phase 4.0B` step numbers.
2. `reports/kil/KIL-DEPENDENCY-GRAPH.md` is the authoritative build-order
   reference for remaining Tier A work.
3. All future KIL implementation commits must cite the `KNW-KIL-DOC-0NN` ID
   and `KIL-NNN` rule range in the commit message, in addition to any
   internal step label.
4. **Known gap, not fixed by this ADR:** `planning.py` lacks the
   no-official-spec disclaimer that `execution.py` and `stack.py` carry. This
   ADR documents the gap; correcting the module docstring is a documentation
   change to existing code and is out of scope for this decision record (no
   production code is changed by this ADR).
5. Tier C modules retain their existing names and step numbers for
   backward compatibility — they are not renamed or deprecated by this ADR.
6. Two already-implemented Tier A documents (Doc 17 Decision, Doc 20
   Thinking) were built ahead of documents they structurally depend on
   (Doc 16 Tradeoffs, Doc 19 Cortex, Doc 08 DNA, etc. — see the Dependency
   Graph). This ADR does not require retrofitting; it flags the ordering
   violation so it is not repeated. New work follows
   `reports/kil/KIL-DEPENDENCY-GRAPH.md`.

---

## Alternatives Considered

- **Renumber all steps to match KIL doc numbers.** Rejected: would rewrite
  git history and invalidate existing `verify.py` suite names
  (`THINK-001..009`, `DEC-001..012`, etc.) that are already relied upon by
  CI-style verification. Classification without renaming preserves history.
- **Treat Tier C modules as informally "part of KIL."** Rejected: they were
  never reviewed against a specification because none exists; conflating
  them with certified KIL coverage would misrepresent actual document
  coverage as higher than it is (7/29 = 24% at the time of this ADR;
  see `reports/kil/KIL-IMPLEMENTATION-MATRIX.md` for the current figure).

---

## Amendments

| Date | Change |
|---|---|
| 2026-07-01 | Docs 01 (Self-Describing), 08 (DNA), 07 (Genome), 02 (Executable) implemented following this ADR's Tier A discipline — each cites its `KNW-KIL-DOC-0NN` ID and `KIL-NNN` rule range directly, with no step-number-only labeling. Document coverage moved from 7/29 (24%) to 11/29 (38%). No changes to the Tier A/B/C classification itself were required. See `reports/kil/KIL-IMPLEMENTATION-MATRIX.md` for full detail. |
