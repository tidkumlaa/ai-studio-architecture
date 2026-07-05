# Platform Kernel — Security Architecture
# Phase 2.0D.2.6A

---

## 1. Overview

The Platform Kernel Security Model enforces a capability-based, deny-by-default
access control system. All access to kernel resources, services, and data
flows through PlatformSecurity.

Security model principles:
- **Deny by default**: All access is denied unless an explicit capability is granted
- **Capability-based**: Access tokens (PlatformCapability) are the unit of authorization
- **Sandbox isolation**: Plugins run in a restricted sandbox, capability-bounded
- **Zero implicit trust**: No component trusts its caller — all calls are checked
- **Audit everything**: Every security decision is recorded in PlatformAudit

---

## 2. Identity Model

### 2.1 Principals

A principal is any entity that can perform actions on the platform.

```python
class PrincipalType(enum.Enum):
    USER    = "user"     # human user
    AGENT   = "agent"    # AI agent
    SERVICE = "service"  # internal service or plugin
    SYSTEM  = "system"   # kernel itself (highest privilege)

@dataclass(frozen=True)
class Principal:
    principal_id: str     # stable, unique identifier
    principal_type: PrincipalType
    display_name: str
    attributes: dict[str, Any]   # role, org, etc.
```

### 2.2 Authentication

```python
class PlatformSecurity:
    def authenticate(self, credentials: "Credentials") -> "AuthenticationResult": ...

@dataclass(frozen=True)
class AuthenticationResult:
    success: bool
    principal: Principal | None
    session_token: str | None
    expires_at: datetime | None
    failure_reason: str | None
    
    def to_context_identity(self) -> str: ...
        # returns principal_id for use in PlatformContext
```

Authentication methods:
- API key (`ApiKeyCredentials`)
- Service token (`ServiceTokenCredentials`) — for plugin-to-kernel auth
- System bootstrap (`SystemCredentials`) — for kernel self-auth at boot

User-facing authentication (OAuth, SAML, SSO) is handled by the
product/desktop layer. The kernel receives an already-authenticated
principal_id via PlatformContext.

### 2.3 Session Token

A session token is issued at authentication and carried in PlatformSession:
- Signed (HMAC-SHA256) to detect tampering
- Contains principal_id and expiry
- Validated on every privileged operation
- Revocable (token revocation list maintained by PlatformSecurity)

---

## 3. Capability System

### 3.1 Capability Definition

```python
@dataclass(frozen=True)
class PlatformCapability:
    capability_id: str     # e.g. "kernel.workspace.read"
    name: str
    description: str
    scope: str             # "kernel" | "plugin" | "user" | "system"
    grantable: bool        # can holders re-grant this to sub-plugins?
```

### 3.2 Capability Grant

```python
@dataclass
class CapabilityGrant:
    grant_id: str
    capability_id: str
    principal_id: str
    granted_by: str        # principal_id of grantor
    granted_at: datetime
    expires_at: datetime | None
    conditions: dict | None    # ABAC conditions (optional constraints)
    revoked: bool = False
```

### 3.3 Grant Hierarchy

```
SYSTEM
  └── kernel.* (all capabilities)
        └── [grants to plugins via PluginManager]
              └── plugin's declared required_capabilities
                    └── [plugin re-grants subset to sub-plugins if grantable=True]
```

A principal can only grant capabilities it itself holds (and where `grantable=True`).
Attempting to grant a capability you don't hold raises `PlatformException(ACCESS_DENIED)`.

### 3.4 Capability Checking

```python
# Check
security.has_capability(principal_id, "kernel.workspace.read")  # bool

# Assert (raises on failure)
security.assert_authorized(
    principal_id=ctx.principal_id,
    action="workspace.read",
    resource_id=workspace_id,
)

# Revoke
security.revoke_capability(principal_id, "kernel.workspace.write")
```

### 3.5 Well-Known Kernel Capabilities

```
kernel.object.read          View any PlatformObject state
kernel.object.dispose       Dispose any PlatformObject

kernel.event.publish        Publish events to PlatformEventBus
kernel.event.subscribe      Subscribe to PlatformEventBus events
kernel.event.replay         Access event replay log

kernel.plugin.load          Load a new plugin
kernel.plugin.unload        Unload a running plugin
kernel.plugin.reload        Hot-reload a running plugin

kernel.config.read          Read configuration values
kernel.config.write         Write runtime configuration overrides

kernel.registry.read        Query PlatformRegistry
kernel.registry.write       Register/unregister objects

kernel.security.admin       Full security administration
kernel.security.grant       Grant capabilities to other principals
kernel.security.revoke      Revoke capabilities
kernel.secrets.*.read       Read a specific named secret (glob)
kernel.secrets.*.write      Write a named secret

kernel.workspace.read       Read workspace metadata
kernel.workspace.write      Create/modify workspaces
kernel.workspace.delete     Delete workspaces

kernel.session.read         Read session data
kernel.session.write        Create/modify sessions
kernel.session.end          End sessions

kernel.execution.read       View execution results
kernel.execution.run        Submit executions

kernel.resource.allocate    Allocate system resources
kernel.resource.quota.admin Modify resource quotas

kernel.diagnostics.read     Read metrics, health, traces
kernel.diagnostics.write    Write custom diagnostic data
kernel.audit.read           Query the audit log
```

---

## 4. Authorization Model

### 4.1 Authorization Flow

```
Request arrives with principal_id, action, resource_id
    │
    ├── Is principal_id authenticated? (session token valid?)
    │       NO → deny, audit: AUTHENTICATION_FAILED
    │
    ├── Does principal hold capability for this action?
    │       NO → deny, audit: CAPABILITY_MISSING
    │
    ├── If resource_id specified, does principal have access to this resource?
    │       (resource-level permission check)
    │       NO → deny, audit: ACCESS_DENIED
    │
    ├── Are ABAC conditions met?
    │       NO → deny, audit: ACCESS_DENIED (with condition details)
    │
    └── ALLOW, audit: access_granted
```

### 4.2 Attribute-Based Access Control (ABAC)

ABAC conditions can restrict capability grants to specific contexts:

```python
# Grant workspace.read only for workspaces owned by the principal
grant = CapabilityGrant(
    capability_id="kernel.workspace.read",
    principal_id="user-alice",
    conditions={
        "workspace.owner_principal": {"equals": "${principal_id}"},
    },
)
```

Condition types: `equals`, `not_equals`, `in`, `starts_with`, `has_tag`.

---

## 5. Plugin Sandbox

### 5.1 Sandbox Design

Every plugin runs inside a PluginSandbox that restricts its access to
only the capabilities declared in its manifest.

```python
class PluginSandbox:
    plugin_id: str
    allowed_capabilities: frozenset[str]
    
    def check(self, action: str, resource_id: str | None = None) -> bool: ...
    
    def __enter__(self) -> "PluginSandbox": ...
    def __exit__(self, *args) -> None: ...
    
    def violation_count(self) -> int: ...
    def last_violation(self) -> dict | None: ...
```

### 5.2 Sandbox Enforcement Points

The sandbox is checked at every kernel API call site:

```python
# In PlatformEventBus.publish():
def publish(self, event: PlatformEvent) -> None:
    if _current_sandbox():
        _current_sandbox().check("kernel.event.publish")
    ...

# In PlatformRegistry.register():
def register(self, obj: PlatformObject, ...) -> RegistrationHandle:
    if _current_sandbox():
        _current_sandbox().check("kernel.registry.write")
    ...
```

### 5.3 Sandbox Violation Handling

When a sandbox violation occurs:
1. The operation is denied (PlatformException raised)
2. Violation is recorded in plugin's sandbox state
3. AuditRecord written: `kernel.security.sandbox_violation`
4. `kernel.security.access.denied` event emitted
5. After `max_violations` (default: 10), plugin is automatically unloaded

---

## 6. Secrets Management

### 6.1 Secrets Store

```python
class SecretsStore:
    def set(
        self,
        name: str,
        value: str,
        owner_id: str,
        *,
        encrypt: bool = True,
        ttl_seconds: int | None = None,
    ) -> None: ...
    
    def get(
        self,
        name: str,
        accessor_id: str,
    ) -> str: ...
        # Checks kernel.secrets.{name}.read capability before returning
    
    def delete(self, name: str, owner_id: str) -> None: ...
    def exists(self, name: str) -> bool: ...
    def list_names(self, accessor_id: str) -> list[str]: ...
        # Returns only names the accessor has read capability for
```

### 6.2 Secret Naming

```
{domain}.{purpose}.{identifier}

platform.api_key.anthropic
platform.api_key.openai
user.{user_id}.personal_token
plugin.{plugin_id}.service_credential
workspace.{workspace_id}.repository_token
```

### 6.3 Secret Access Audit

Every call to `get()` is audited:
```
action: "kernel.secrets.accessed"
details: {name: "platform.api_key.anthropic", accessor_id: "..."}
```

Secret VALUES are never written to audit or logs.

---

## 7. Encryption

The kernel does not implement encryption primitives. It delegates to
`platform_sdk.security` for:
- Symmetric encryption of secrets at rest (AES-256-GCM)
- HMAC signing of session tokens
- Hashing of sensitive values (bcrypt for passwords, if applicable)

```python
class EncryptionService:
    def encrypt(self, plaintext: bytes, key_id: str) -> bytes: ...
    def decrypt(self, ciphertext: bytes, key_id: str) -> bytes: ...
    def sign(self, data: bytes, key_id: str) -> str: ...
    def verify(self, data: bytes, signature: str, key_id: str) -> bool: ...
    def hash(self, value: str) -> str: ...
```

Key management:
- Keys are stored in the SecretsStore under `platform.encryption.key.*`
- Key rotation is manual for Phase 2.0D.2.6A (automated rotation in later phase)

---

## 8. Security Events

```
kernel.security.access.granted     principal_id, action, resource_id
kernel.security.access.denied      principal_id, action, resource_id, reason
kernel.security.capability.granted principal_id, capability_id, granted_by
kernel.security.capability.revoked principal_id, capability_id, revoked_by
kernel.security.sandbox.violation  plugin_id, action, resource_id
kernel.security.secret.accessed    accessor_id, secret_name (NOT value)
kernel.security.token.issued       principal_id, expires_at
kernel.security.token.revoked      principal_id, reason
kernel.security.auth.success       principal_id, auth_method
kernel.security.auth.failure       credential_type, reason (NOT credentials)
```

All security events carry `priority=HIGH` and `persistent=True`.

---

## 9. Security Invariants

1. The SYSTEM principal is never passed through plugin code — it is
   kernel-internal only.
2. A capability cannot be granted by a principal who does not hold it.
3. Secret values are encrypted at rest and never appear in logs or events.
4. PluginSandbox is always active during plugin code execution (no bypass).
5. Authentication failure rate limiting: > 10 failures/minute per credential
   type triggers a 5-minute cooldown and `kernel.security.auth.lockout` event.
6. Session tokens expire after 8 hours of inactivity (configurable).
7. Capability grants cannot be backdated (expires_at must be future or None).
