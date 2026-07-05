# ⚠️ HISTORICAL REPOSITORY — SUNSET 2026-07-05

This repository (`ai-studio-architecture`) is **frozen**. Its full content
and history now live inside the main AI Studio repository at
`docs/archive/architecture-import/` (imported by `git subtree add`, so
`git log --follow` there reaches every commit made here).

## Why it was sunset

This repo held the 2026-06 architecture corpus: 501 documents across six
naming generations (1.5.5, 2.0, 3.0, 3.5, kos, kos-final, knowledge-*).
The corpus is genuinely valuable history, but as a separate submodule it
had no CI, drifted from the code it described, and split every search and
AI-context compilation. **ADR-0002** in the main repository (repository
strategy: product monorepo — ratified by the Maintainer 2026-07-05)
consolidated it into the monorepo's archive, superseding the 2026-07-05
submodule topology via that decision's own successor clause.

## Replacement & migration path

- **Canonical architecture documentation:** the main repo's
  `docs/constitution/`, `docs/architecture/`, and `docs/adr/` —
  governed by `docs/GOVERNANCE-INDEX.md` with machine-checked statuses.
- **This corpus:** `docs/archive/architecture-import/` in the main repo,
  registered *en bloc* as `historical`; per-document triage (promotion of
  anything still canonical) is the E3 workstream there.
- Nothing external ever consumed this repo programmatically; no code
  migration exists or is needed.

## Final state

- **Final content commit:** `c0b76fc` (`docs(architecture): add knowledge,
  runtime, migration, kernel & SDK docs`)
- **Terminal marker:** this commit, tagged **`sunset-2026-07-05`**
- **Historical purpose:** the written record of AI Studio's architectural
  thinking, June–July 2026 — six generations of it, preserved exactly
  because the Constitution (Article XIII) forbids destroying the record
  of how the thinking evolved.
- **Status:** historical repository, read-only by policy; GitHub archiving
  deferred until AI Studio 1.0 by Maintainer decision (2026-07-05).

Main repository: https://github.com/tidkumlaa/ai-studio-platform-kernel
