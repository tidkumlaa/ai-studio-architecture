# KNW-FINAL-007 — Canonical Workflows

**Phase:** 3.0D.0.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Defines the standard end-to-end workflows for common KOS operations — object creation, promotion, deprecation, package management, and certification.

---

## WF-001 — Create and Promote a New Object

```
1. Author creates YAML from template
   kos template create module --name "My Module" --namespace plt
   → Generates knw-plt-mod-NNN.yaml with DRAFT state

2. Fill required fields
   - description, owner, responsibilities, traceability.satisfies
   
3. Format
   kos format knw-plt-mod-NNN.yaml

4. Lint
   kos lint knw-plt-mod-NNN.yaml
   → Fix any KL-* violations

5. Propose
   kos lifecycle propose KNW-PLT-MOD-NNN
   → State: DRAFT → PROPOSED
   → Gate: schema valid + owner set

6. Open PR
   git add + git commit + git push + PR opened
   → CI: Core certification runs (10 min)

7. Review
   1 reviewer approval required

8. Merge
   → CI: Merge gate certification (2 hours)

9. Verify
   kos lifecycle verify KNW-PLT-MOD-NNN
   → Gate: kos lint pass + quality ≥ 0.60
   → State: PROPOSED → VERIFIED

10. Approve for CANONICAL
    kos lifecycle approve KNW-PLT-MOD-NNN
    → Gate: quality ≥ 0.80 + evidence complete
    → State: VERIFIED → CANONICAL
```

**Total time (typical):** 2–5 days (review + quality iteration)

---

## WF-002 — Deprecate and Replace an Object

```
1. Create successor object (follow WF-001)
   KNW-PLT-MOD-NNN (new version or replacement)
   → Promote to VERIFIED

2. Deprecate old object
   kos lifecycle deprecate KNW-PLT-MOD-OLD --successor KNW-PLT-MOD-NNN
   → Sets deprecated_at, successor_id, sunset_at = deprecated_at + 90 days
   → State: CANONICAL → DEPRECATED

3. Notify dependents
   kos notify --deprecated KNW-PLT-MOD-OLD
   → All packages with DEPENDS_ON KNW-PLT-MOD-OLD are notified

4. Migration period (90 days)
   Dependent packages update to use successor

5. Archive after sunset
   kos lifecycle archive KNW-PLT-MOD-OLD
   → State: DEPRECATED → ARCHIVED
   → Object no longer returned in default queries
```

---

## WF-003 — Add a New Package

```
1. Architecture Board approval (if new domain)
   kos adm new-package --namespace new --domain NEW

2. Create package.yaml
   kos package init kos.new.package

3. Add to workspace.yaml
   - kos.new.package

4. Create 5+ seed objects
   kos template create module --namespace new --count 5

5. Install package
   kos install kos.new.package

6. Lock
   kos lock

7. Run certification
   kos-cert registry
   kos-cert dependency

8. Open PR with package + seed objects
```

---

## WF-004 — Run Certification

```
# Pre-commit (automatic on commit)
kos-cert verify --stage pre-commit

# PR check (automatic on PR open)
kos-cert verify --stage pr

# Manual full certification
kos-cert verify --level gold --seed 42 --output reports/cert-$(date +%Y%m%d)/

# Check result
kos-cert dashboard

# Compare to previous
kos-cert evolution --baseline reports/cert-prev/ --current reports/cert-now/
```

---

## WF-005 — Update Evidence on an Object

```
1. Identify stale evidence
   kos evidence audit --stale --package kos.platform.package

2. Gather new evidence
   (run benchmark, get test result, write doc)

3. Add evidence record to object YAML
   evidence:
     items:
       - evidence_type: EV-BENCHMARK
         source_id: KNW-TEST-BENCH-001
         description: "Benchmark run 2026-06-30"
         weight: 0.70
         freshness_score: 1.0
         captured_at: "2026-06-30T00:00:00Z"

4. Format + lint
   kos format + kos lint

5. Recompute quality score
   kos quality score KNW-PLT-MOD-001
   → Should increase due to fresh evidence

6. Open PATCH PR (1 reviewer, no board approval needed)
```

---

## WF-006 — Build and Publish Golden Dataset

```
1. Ensure all 9 packages installed
   kos install --all --frozen

2. Generate dataset
   kos dataset build --size 100K --seed 42

3. Generate ground truth
   kos dataset ground-truth --queries 10000 --reasoning 5000

4. Verify dataset integrity
   kos-cert dataset --verify

5. Publish manifest
   kos dataset publish --output knowledge/datasets/manifest.yaml
```

---

## WF-007 — Rename an Object (MAJOR change)

```
1. Create ADR documenting rename rationale
   KNW-META-DEC-NNN: "Rename KNW-PLT-MOD-OLD → KNW-PLT-MOD-NEW"

2. Architecture Board approval

3. Create new object with new ID and canonical_name

4. Add alias: old ID → new ID
   kos alias add KNW-PLT-MOD-OLD KNW-PLT-MOD-NEW

5. Update all references in other objects
   kos refactor --rename KNW-PLT-MOD-OLD KNW-PLT-MOD-NEW --dry-run
   kos refactor --rename KNW-PLT-MOD-OLD KNW-PLT-MOD-NEW

6. Deprecate old object
   kos lifecycle deprecate KNW-PLT-MOD-OLD --successor KNW-PLT-MOD-NEW

7. Keep alias active ≥ 90 days

8. Bump package version MAJOR
```

---

## Cross-References

- Lifecycle states → Phase 3.0C.5 `19-KNOWLEDGE-LIFECYCLE`
- Change management → Phase 3.0C.5 `25-KNOWLEDGE-CHANGE-MANAGEMENT`
- CI stages → Phase 3.0D.0 `29-CI-INTEGRATION`
- Evidence patterns → `13-CANONICAL-EVIDENCE`
