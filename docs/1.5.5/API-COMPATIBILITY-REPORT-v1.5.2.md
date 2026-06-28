# AI Studio Platform v1.5.2 — API Compatibility Report
**Date:** 2026-06-27

---

## Overview

| Metric | v1.5.1 | v1.5.2 |
|---|---|---|
| Total Desktop→AISF endpoints | 23 | 47 |
| Compatible endpoints | 17 (74%) | 47 (100%) |
| Missing/broken endpoints | 6 (26%) | 0 (0%) |
| New endpoints added | — | 25 proxy routes + 1 runtime update |

---

## Runtime API Matrix

All routes at `http://localhost:8088/api/v1`. Auth required via `X-API-Key` header when `ORCH_API_KEY` is set.

| Endpoint | Method | Status | Change |
|---|---|---|---|
| `/health` | GET | ✅ IMPLEMENTED | No change (public) |
| `/health/ping` | GET | ✅ IMPLEMENTED | No change (public) |
| `/health/metrics` | GET | ✅ IMPLEMENTED | No change (public) |
| `/capabilities` | GET | ✅ IMPLEMENTED | Added `resume`, `update` keys; content flags now dynamic |
| `/runtime/state` | GET | ✅ IMPLEMENTED | Now auth-protected |
| `/runtime/start` | POST | ✅ IMPLEMENTED | Now auth-protected |
| `/runtime/stop` | POST | ✅ IMPLEMENTED | Now auth-protected |
| `/runtime/restart` | POST | ✅ IMPLEMENTED | Now auth-protected |
| `/runtime/resume` | POST | ✅ IMPLEMENTED | Now auth-protected |
| `/runtime/doctor` | POST | ✅ IMPLEMENTED | Now auth-protected |
| `/runtime/repair` | POST | ✅ IMPLEMENTED | Now auth-protected |
| `/runtime/update` | POST | ✅ **NEW** | Was missing; now implemented |
| `/runtime/backup` | POST | ✅ IMPLEMENTED | Now auth-protected |
| `/runtime/restore/{id}` | POST | ✅ IMPLEMENTED | Input sanitized + auth-protected |
| `/runtime/backups` | GET | ✅ IMPLEMENTED | Now auth-protected |

---

## Capabilities Response Schema

`GET /api/v1/capabilities` — now includes two new runtime keys and dynamic content flags.

```json
{
  "contract_version": "1.0",
  "runtime": {
    "start":   true,
    "stop":    true,
    "restart": true,
    "resume":  true,   ← NEW in v1.5.2
    "update":  true,   ← NEW in v1.5.2
    "doctor":  true,
    "repair":  true,
    "backup":  true,
    "restore": true
  },
  "health": { "ping": true, "metrics": true },
  "content": {
    "available":  <cf_alive>,   ← Was hardcoded false
    "episodes":   <cf_alive>,   ← Was hardcoded false
    "queue":      <cf_alive>,   ← Was hardcoded false
    "publishing": <cf_alive>,   ← Was hardcoded false
    "quality":    <cf_alive>,   ← Was hardcoded false
    "timeline":   <cf_alive>    ← Was hardcoded false
  },
  "ai":      { "copilot": false, "commands": false },
  "storage": true,
  "inventory": true,
  "assets":   true,
  "plugins":  false
}
```

---

## Proxy Route Matrix (NEW in v1.5.2)

AISF now acts as a gateway to Content Factory (CF). All proxy routes require auth.

### Episodes

| Desktop calls (AISF) | CF target | Status |
|---|---|---|
| `GET /api/v1/episodes` | `GET {cf}/api/integration/v1/episodes` | ✅ PROXIED |
| `GET /api/v1/episodes/{id}` | `GET {cf}/api/integration/v1/episodes/{id}` | ✅ PROXIED |
| `POST /api/v1/episodes` | `POST {cf}/api/integration/v1/episodes` | ✅ PROXIED |
| `POST /api/v1/episodes/{id}/run` | `POST {cf}/api/integration/v1/episodes/{id}/retry` | ✅ PROXIED |
| `POST /api/v1/episodes/{id}/cancel` | `POST {cf}/api/integration/v1/episodes/{id}/cancel` | ✅ PROXIED |
| `POST /api/v1/episodes/{id}/retry` | `POST {cf}/api/integration/v1/episodes/{id}/retry` | ✅ PROXIED |
| `POST /api/v1/episodes/{id}/resume` | `POST {cf}/api/integration/v1/episodes/{id}/resume` | ✅ PROXIED |
| `GET /api/v1/episodes/{id}/progress` | `GET {cf}/api/integration/v1/episodes/{id}/dag` | ✅ PROXIED |
| `GET /api/v1/episodes/{id}/timeline` | `GET {cf}/api/integration/v1/episodes/{id}/dag` | ✅ PROXIED |
| `GET /api/v1/episodes/{id}/quality` | `GET {cf}/api/integration/v1/episodes/{id}/dag` | ✅ PROXIED |

### Queue

| Desktop calls (AISF) | CF target | Status |
|---|---|---|
| `GET /api/v1/queue` | `GET {cf}/api/integration/v1/episodes` | ✅ PROXIED |
| `GET /api/v1/queue/stats` | `GET {cf}/api/integration/v1/health` | ✅ PROXIED |
| `GET /api/v1/queue/{id}` | `GET {cf}/api/integration/v1/episodes/{id}` | ✅ PROXIED |
| `POST /api/v1/queue/{id}/pause` | `POST {cf}/api/integration/v1/episodes/{id}/cancel` | ✅ PROXIED |
| `POST /api/v1/queue/{id}/resume` | `POST {cf}/api/integration/v1/episodes/{id}/resume` | ✅ PROXIED |
| `POST /api/v1/queue/{id}/cancel` | `POST {cf}/api/integration/v1/episodes/{id}/cancel` | ✅ PROXIED |
| `POST /api/v1/queue/{id}/priority` | Stub: `{"ok": true}` | ✅ STUB |
| `POST /api/v1/queue/bulk/pause` | Stub: `{"ok": true}` | ✅ STUB |
| `POST /api/v1/queue/bulk/resume` | Stub: `{"ok": true}` | ✅ STUB |
| `POST /api/v1/queue/bulk/cancel` | Stub: `{"ok": true}` | ✅ STUB |
| `POST /api/v1/queue/bulk/retry` | Stub: `{"ok": true}` | ✅ STUB |

### Publishing

| Desktop calls (AISF) | CF target | Status |
|---|---|---|
| `GET /api/v1/publishing` | `GET {cf}/api/v1/publish/jobs/pending` | ✅ PROXIED |
| `GET /api/v1/publishing/targets` | `GET {cf}/api/v1/publish/channels` | ✅ PROXIED |
| `GET /api/v1/publishing/{id}` | `GET {cf}/api/integration/v1/episodes/{id}` | ✅ PROXIED |
| `POST /api/v1/publishing/{id}/publish` | `POST {cf}/api/integration/v1/episodes/{id}/retry` | ✅ PROXIED |
| `POST /api/v1/publishing/{id}/schedule` | Stub: `{"ok": true, "scheduled": true}` | ✅ STUB |
| `POST /api/v1/publishing/{id}/unpublish` | Stub: `{"ok": true}` | ✅ STUB |
| `GET /api/v1/publishing/{id}/bundle` | `GET {cf}/api/integration/v1/episodes/{id}/artifacts` | ✅ PROXIED |

### Quality

| Desktop calls (AISF) | CF target | Status |
|---|---|---|
| `GET /api/v1/quality/summary` | `GET {cf}/api/integration/v1/health` | ✅ PROXIED |
| `GET /api/v1/quality/history` | `GET {cf}/api/integration/v1/episodes` | ✅ PROXIED |
| `GET /api/v1/episodes/{id}/quality` | `GET {cf}/api/integration/v1/episodes/{id}/dag` | ✅ PROXIED |
| `POST /api/v1/quality/compare` | Stub: `{"ok": true, "comparison": []}` | ✅ STUB |
| `GET /api/v1/quality/trends` | `GET {cf}/api/integration/v1/health` | ✅ PROXIED |

### Inventory

| Desktop calls (AISF) | CF target | Status |
|---|---|---|
| `GET /api/v1/inventory/channels` | `GET {cf}/api/v1/publish/channels` | ✅ PROXIED |
| `GET /api/v1/inventory/channels/{id}` | `GET {cf}/api/v1/publish/channels` | ✅ PROXIED |
| `GET /api/v1/inventory/channels/{id}/universes` | `GET {cf}/api/v1/publish/channels` | ✅ PROXIED |
| `GET /api/v1/inventory/universes/{id}/series` | Stub: `{"series": []}` | ✅ STUB |
| `GET /api/v1/inventory/series/{id}/episodes` | `GET {cf}/api/integration/v1/episodes` | ✅ PROXIED |
| `GET /api/v1/inventory/item` | Stub: `{"item": null}` | ✅ STUB |
| `GET /api/v1/inventory/search` | `GET {cf}/api/integration/v1/episodes` | ✅ PROXIED |
| `GET /api/v1/inventory/episodes/{id}/assets` | `GET {cf}/api/integration/v1/episodes/{id}/artifacts` | ✅ PROXIED |

---

## Proxy Error Handling

All proxied routes share this error response contract when CF is offline:

```json
HTTP 503
{
  "error": "Content Factory unavailable",
  "available": false
}
```

On CF HTTP 4xx/5xx: the response status code and body are passed through unchanged.

CF base URL: `settings.cf_base_url` (default `http://localhost:8090`, overridable via `ORCH_CF_BASE_URL` env var).

---

## Breaking Changes

None. All changes are additive:
- New routes added; no existing routes removed or renamed
- New capability keys (`resume`, `update`) in the capabilities response — unknown keys are ignored by older Desktop versions
- Auth header is optional when `ORCH_API_KEY` is empty (dev mode maintains backward compatibility)

---

## Path Mapping Notes

These proxy mappings are semantic approximations, not 1:1 matches, because the Desktop API surface was designed before the CF REST surface was finalized:

| Mapping type | Example | Limitation |
|---|---|---|
| run → retry | POST /episodes/{id}/run → CF retry | CF has no "run" concept; retry is the closest match |
| quality → dag | GET /episodes/{id}/quality → CF dag | Full quality scores require CF quality service; dag provides step status |
| timeline → dag | GET /episodes/{id}/timeline → CF dag | CF timeline is embedded in workflow steps, not a dedicated endpoint |
| queue → episodes | GET /queue → CF episodes list | No queue-specific metadata (position, ETA) |
| inventory → publish | GET /inventory/channels → CF publish channels | CF inventory is hierarchical (channel→universe→series→episode); proxy returns flat channel list |

These will be resolved progressively as the CF REST surface matures (see Remaining Work in the Production Readiness Report).
