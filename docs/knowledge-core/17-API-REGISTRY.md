# KNW-KC-ARCH-017 — API Registry

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The API Registry catalogs all API surfaces exposed by the platform and its runtimes — REST endpoints, Python protocols, CLI commands, gRPC services, and MCP interfaces. Every API is versioned and has a discoverable contract.

---

## Registered API Families

| Family | Base Path / Entry Point | Type | Version |
|--------|------------------------|------|---------|
| Intelligence API | `/api/v1/intelligence/` | REST | v1 |
| Resources API | `/api/v1/resources/` | REST | v1 |
| Providers API | `/api/v1/providers/` | REST | v1 |
| Platform API | `/api/v1/platform/` | REST | v1 |
| Knowledge API | `/api/v1/knowledge/` | REST | v1 (planned) |
| Platform CLI | `platform-*` commands | CLI | v1 |
| KOS CLI | `platform-kos *` commands | CLI | v1 |
| Python SDK | `platform_sdk.*` modules | Python | v2 |

---

## API Entry Format

```yaml
# knowledge/registry/apis/intelligence-api-v1.yaml
identity:
  knowledge_id: "KNW-API-REST-intelligence-v1"
  canonical_name: "api.rest.intelligence.v1"
  knowledge_uri: "knw://api/rest/intelligence-v1"
  namespace: "api"
  version: "1.0.0"
  owner: "team:platform-core"

api_spec:
  api_type: REST
  base_path: "/api/v1/intelligence"
  versioned: true
  auth_required: true
  rate_limited: true
  openapi_path: "architecture/specs/intelligence-api-v1.yaml"

  endpoints:
    - path: "/execute"
      method: POST
      handler: "api.intelligence.execute_workload"
      request_schema: "ExecutionRequest"
      response_schema: "ExecutionResponse"
      latency_budget_ms: 30000

    - path: "/stream"
      method: POST
      handler: "api.intelligence.stream_workload"
      request_schema: "StreamRequest"
      response_schema: "StreamChunk (streaming)"
      latency_budget_ms: null   # streaming — no fixed budget

    - path: "/predict"
      method: POST
      handler: "api.intelligence.predict"
      request_schema: "PredictionRequest"
      response_schema: "PredictionResponse"
      latency_budget_ms: 200

  standard_errors:
    400: "ValidationError"
    401: "AuthenticationRequired"
    403: "PermissionDenied"
    404: "ResourceNotFound"
    429: "RateLimitExceeded"
    500: "InternalError"
    503: "ServiceUnavailable"

lifecycle:
  status: CANONICAL
```

---

## API Versioning Rules

### AR-001 — Version in Path
REST APIs use URL versioning: `/api/v{N}/`. No header-based versioning.

### AR-002 — Breaking Change Policy
MAJOR version bump required for:
- Removing or renaming an endpoint
- Changing a required request field
- Changing a response field's type
- Removing a response field

### AR-003 — Deprecation Notice
Deprecated API endpoints carry a `Deprecation` header and a sunset date. Minimum 90-day sunset period.

### AR-004 — Contract First
Every API endpoint must have an OpenAPI spec or equivalent before implementation begins. No implementation without spec.

### AR-005 — Error Consistency
All endpoints use the standard error envelope:
```json
{
  "error": "RESOURCE_NOT_FOUND",
  "message": "Knowledge object KNW-KOS-ARCH-999 not found",
  "request_id": "req-abc123",
  "timestamp": "2026-06-01T12:00:00Z",
  "docs_uri": "knw://api/errors/RESOURCE_NOT_FOUND"
}
```

---

## API Registry Protocol

```python
class APIRegistry(KnowledgeRegistryProtocol):
    def get_by_path(self, base_path: str) -> APIObject: ...
    def get_by_type(self, api_type: str) -> list[APIObject]: ...
    def get_endpoints(self, api_id: str) -> list[APIEndpoint]: ...
    def get_spec(self, api_id: str) -> dict: ...                  # OpenAPI dict
    def get_deprecated(self) -> list[APIObject]: ...
    def validate_contract(self, api_id: str) -> ContractValidationResult: ...
    def get_consumers(self, api_id: str) -> list[str]: ...
```

---

## API Discovery Index

```yaml
# indexes/api-index.yaml
rest:
  "/api/v1/intelligence": "KNW-API-REST-intelligence-v1"
  "/api/v1/resources": "KNW-API-REST-resources-v1"
  "/api/v1/providers": "KNW-API-REST-providers-v1"
  "/api/v1/platform": "KNW-API-REST-platform-v1"
  "/api/v1/knowledge": "KNW-API-REST-knowledge-v1"
cli:
  "platform-verify": "KNW-API-CLI-platform-verify"
  "platform-migrate": "KNW-API-CLI-platform-migrate"
  "platform-kos": "KNW-API-CLI-kos-root"
python:
  "platform_sdk": "KNW-API-PYTHON-platform-sdk-v2"
```

---

## Cross-References

- API objects in Object Registry → `15-OBJECT-REGISTRY`
- API endpoints in Phase 2.1D.0 → `30-API.md`
- CLI spec in Phase 2.1D.0 → `29-CLI.md`
- Service discovery (APIs built on services) → `16-SERVICE-REGISTRY`
