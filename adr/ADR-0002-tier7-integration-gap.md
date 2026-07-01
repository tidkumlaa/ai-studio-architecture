# ADR-0002 — Tier-7 Engine / Typed KIL Block Integration Gap

**Status:** Accepted (documenting a known, deliberate gap — not a decision to resolve it now)
**Date:** 2026-07-01
**Owner:** architecture-board
**Related:** ADR-0001 (KIL phase classification), `reports/kil/KIL-IMPLEMENTATION-MATRIX.md`, `reports/kil/KIL-DEPENDENCY-GRAPH.md`

---

## Context

During the impact assessment for KIL Document 04 (AI Context Layer), the
data-flow of the five already-certified Tier-7 engines was traced:

```
air_score.py   : meta = node.metadata or {}   → _w2_ai_context(meta) reads meta["short_summary"] etc.
kil_search.py  : meta = node.metadata or {}   → embedding_fields reads meta["search"]
kil_index.py   : meta = node.metadata or {}   → short_summary, nano, dna_hash, genome(raw) all from meta
kil_metrics.py : meta = node.metadata or {}   → KIM-003/015 read meta["short_summary"] etc.
dashboard.py   : same generic content-dict pattern via _present()
```

All five read exclusively from `KnowledgeNode.metadata` — a generic,
untyped, free-form `dict[str, Any]` field.

Separately, the docs implemented under this initiative (01 Self-Describing,
07 Genome, 08 DNA, 02 Executable, 04 AI Context) each store their data in a
**typed, validated block** attached via a dedicated `KIGRuntime` store:

```
kig.self_describing_for(node_id) -> SelfDescribingBlock | None
kig.dna_for(node_id)             -> DnaBlock | None
kig.genome_for(node_id)          -> GenomeBlock | None
kig.executable_for(node_id)      -> ExecutableBlock | None
kig.ai_context_for(node_id)      -> AiContextBlock | None
```

**These two data paths do not currently connect.** A node can have a fully
valid, KIL-compliant `AiContextBlock` attached via `kig.add_ai_context()`
and the AIRS engine's W2 dimension will still score it as empty, because
`air_score.py` never looks at `kig.ai_context_for()` — it only reads
`node.metadata`. The same is true for W1 (self_describing), W7 (genome),
and W8 (dna) relative to the docs already implemented, and for
`kil_search.py`'s embedding source fields, `kil_index.py`'s index entry
fields, `kil_metrics.py`'s KIM completeness metrics, and `dashboard.py`'s
block-completion matrix.

## Decision

**No integration is performed as part of implementing docs 01, 02, 04, 07,
or 08.** Each of these documents was implemented as a strictly additive,
standalone module with zero modification to the five certified Tier-7
engines, matching the scope explicitly agreed for this phase.

This gap is deliberate, not an oversight: modifying `air_score.py`,
`kil_search.py`, `kil_index.py`, `kil_metrics.py`, and `dashboard.py` means
touching five modules with their own certified verify suites
(`AIRS-001..020`, `SRCH-001..020`, `CIDX-001..020`, `KIM-001..020`,
`DASH-001..015`). Bundling that into a foundation-document implementation
would both violate the "implement only the official specification" scoping
used for every doc so far, and multiply review/regression risk in a single
change.

## Consequences

1. Implementing more foundation docs (04 now; 03, 05, 06, 09, 13, 19, etc.
   later) will **not** move AIRS/Search/Index/Metrics/Dashboard scores or
   coverage numbers, even though the underlying typed data exists. Anyone
   reading those reports should not expect them to reflect docs 01/02/04/07/08
   until the integration below happens.
2. A future **"Tier-7 Integration Pass"** is planned: once a sufficient set
   of foundation documents exist (candidates per the Dependency Graph: 01,
   03, 04, 06, 07, 08, 13, 19 — the docs that directly feed W1–W8 of the
   AIRS formula), the five Tier-7 engines will be updated to read from the
   typed `kig.X_for()` stores first, falling back to `node.metadata` when
   no typed block is attached. This preserves every object that still only
   populates `node.metadata` (no regression) while giving typed-block
   objects their real scores.
3. This ADR does not schedule that pass. It is raised for architecture-board
   awareness so the gap is never mistaken for a defect in docs 01/02/04/07/08
   — those documents are complete and verified against their own spec; the
   gap lives entirely in the five *older* engines not yet knowing the new
   stores exist.
4. `reports/kil/KIL-IMPLEMENTATION-MATRIX.md` §3 (Schema Coverage) already
   carries a per-doc note about this; this ADR is the canonical, single
   place documenting the systemic cause across all five engines at once.

## Alternatives Considered

- **Update the five engines now, incrementally, one per foundation doc.**
  Rejected for this phase: five separate blast-radius expansions across
  already-certified modules is a larger, riskier unit of work than the
  explicit "implement doc 04 only" scope approved for this session.
- **Have new engines write into `node.metadata` instead of a typed store**
  (so old engines pick them up "for free"). Rejected: this would silently
  discard the type safety, immutability (DNA), and validation-rule
  enforcement that are the entire point of implementing these as typed
  `KIGRuntime` stores rather than free-form dict entries — it would trade
  away real correctness guarantees to avoid a known, well-scoped future task.
