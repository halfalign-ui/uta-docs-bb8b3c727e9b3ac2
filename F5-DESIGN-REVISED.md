# F5 — CLOUD EXECUTION & AGENT RUNTIME

## DESIGN REVISED: 2026-08-17
## STATUS: DESIGN REVIEW COMPLETE / IMPLEMENTATION PENDING APPROVAL

---

## A. Current F5 Preflight Findings

### Filesystem State

```
/secondary/              229G ext4, 625MB used (1%)
├── backups/             empty
├── cache/               phase8e/8f/8g/9 benchmark data (vox-owned, ROOT must not touch)
├── logs/                empty
└── lost+found/          root-only
```

`/secondary/cache/` contains benchmark data from previous work. MUST remain untouched.

### Vault State

- `/data/vault/` exists on vox-space, owned `vault:vault`, mode 0700
- Currently EMPTY — no secrets stored
- F5 does NOT modify vault contents. F5 reads credentials through F4 MCP boundary only.

### Existing Credentials

| Credential | Location | Owner | Used By |
|---|---|---|---|
| `AI_GATEWAY_API_KEY` | `/etc/ai-gateway/gateway.env` | root:0600 | ai-gateway.service |
| `local.key` | `/home/vox/.config/opencode/local.key` | vox:0600 | opencode CLI |

F5 does NOT migrate these credentials. That is a separate approval-gated task.

### Running Services

- `ai-gateway.service` — local inference proxy (127.0.0.1:8090 → llama-server)
- `llama-server.service` — Qwen2.5-7B Q4_K_M local inference
- `portal-bot.service` — Telegram bot
- Docker, containerd, nvidia-persistenced

F5 does NOT modify any existing service.

---

## B. Changes from Original F5 Proposal

| Area | Original Proposal | Revised Design | Reason |
|---|---|---|---|
| Trust model | "NETWORK capability, no approval" | Capability + approval gated per provider | Deny-by-default requires explicit allow per provider |
| Credential handling | "flush from memory" | Lifetime-scoped, no cache, no filesystem | Python cannot reliably zero memory |
| Provider config | Python dataclass | JSON config file (stdlib) | No code changes for provider registration |
| Network boundary | Implicit | Explicit: endpoint allowlist, TLS mandatory, no arbitrary URLs | Arbitrary HTTP is a bypass vector |
| MCP tools | 4 tools (execute, list_providers, status, quota) | 5 tools (execute, list_providers, execution_status, quota_status, cancel) | Cancellation needed for lifecycle |
| Runtime storage | Metadata + response bodies | Metadata-only by default | Response bodies may contain sensitive content |
| Streaming | Yes | Yes, with explicit boundary | Must not become unbounded buffer |
| Fallback | Priority-based | Priority-based, policy-checked per attempt | Fallback must not bypass policy |
| Quota | Single tracker | Per-provider, per-model, per-period | Providers have different billing models |
| Scope | "Cloud execution only" | Cloud execution + credential resolution + quota | Narrow scope prevents creep |

---

## C. Final F5 Architecture

### Layer Position

```
INGRESS (POST /command, /mcp, /cloud)
  ↓
AUTH (token validation)
  ↓
POLICY (deny-by-default, role + capability + resource)
  ↓
APPROVAL (if required by policy rule)
  ↓
CLOUD EXECUTION (F5 boundary)
  ├─ ProviderRegistry → JSON config
  ├─ CredentialResolver → F4 Vault MCP (never direct filesystem)
  ├─ CloudExecutor → HTTP to provider
  ├─ QuotaTracker → /secondary/logs/quota.json
  └─ ExecutionLifecycle → state machine
  ↓
AUDIT (append-only, hash-chain, secret values redacted)
  ↓
RESULT (egress filter → caller)
```

### Module Layout

```
gate/cloud/
    __init__.py
    provider.py        # ProviderConfig, ModelSpec, CapabilitySpec dataclasses
    registry.py        # ProviderRegistry: load from JSON, lookup by ID
    credential.py      # CredentialResolver: vault MCP → api_key (ephemeral)
    executor.py        # CloudExecutor: HTTP call to provider
    lifecycle.py       # ExecutionStateMachine: PENDING → DONE
    quota.py           # QuotaTracker: per-provider/model accounting
    streaming.py       # StreamingProxy: provider SSE → caller SSE
    config.py          # CloudConfig: load from JSON
    errors.py          # F5-specific errors
```

---

## D. Trust / Security Model

### Trust Boundaries (F5 additions)

```
Cloud L2 (UNTRUSTED)
  ↓ JSON command
L3 INGRESS (auth + schema)
  ↓ authenticated request
POLICY ENGINE (deny-by-default)
  ↓ authorized request
APPROVAL GATE (if required)
  ↓ approved request
F5 CLOUD EXECUTION (new boundary)
  ├─ provider validation (is this provider allowed?)
  ├─ model validation (is this model allowed for this provider?)
  ├─ credential resolution (vault MCP, ephemeral)
  ├─ HTTP call (TLS to known endpoint only)
  ├─ response validation (is response well-formed?)
  └─ egress filter (redact secrets from response)
  ↓
AUDIT (immutable log)
```

### F5 Security Rules

1. **F5 is NOT an HTTP proxy.** It only calls pre-registered provider endpoints. No arbitrary URLs.
2. **F5 does NOT store credentials.** Credentials are resolved from vault, used ephemerally, discarded.
3. **F5 does NOT bypass policy.** Every cloud execution passes through the full policy pipeline.
4. **F5 does NOT bypass approval.** If policy says approval required, cloud execution waits.
5. **F5 does NOT log secrets.** Credential values never appear in audit, logs, or error messages.
6. **F5 does NOT trust provider responses blindly.** Responses are validated before returning to caller.
7. **F5 does NOT allow prompt injection through responses.** Egress filter applies.

### Policy Capability

Cloud execution uses `NETWORK` capability. This is the existing capability class in `gate/policy/classes.py`:

```python
class Capability(str, Enum):
    READ = "READ"
    WRITE = "WRITE"
    EXECUTE = "EXECUTE"
    SECRET = "SECRET"
    NETWORK = "NETWORK"    # ← F5 uses this
    BACKGROUND = "BACKGROUND"
```

**Approval requirement:** Policy rules determine whether a specific provider/model requires approval. This is NOT hardcoded — it is a policy decision:

```python
# Example policy rules for F5:
Rule("admin", "NETWORK", "cloud.execute", allow=True, resource="cloud:openrouter")
Rule("admin", "NETWORK", "cloud.execute", allow=True, require_approval=True, resource="cloud:anthropic")
Rule("user",  "NETWORK", "cloud.execute", allow=True, resource="cloud:openrouter")
# user has NO rules for anthropic → deny by default
```

**Why admin-only for some providers:** Cost control and risk management. A user role executing expensive model calls without approval is a financial risk. Policy enforces this, not F5 code.

**Why no approval for allowed providers:** If policy explicitly allows a provider+model combination for a role, requiring additional approval adds friction without security benefit. The policy rule IS the approval — it was set by an admin. Per-execution approval is for operations that are inherently dangerous (file writes, vault writes). Cloud inference is not inherently dangerous — it is an outbound HTTP call to a known, trusted endpoint.

**Preventing arbitrary outbound HTTP:** The `resource` field in policy rules uses a `cloud:{provider_id}` namespace. F5 validates that the requested provider_id exists in the registry BEFORE policy evaluation. If the provider is not registered, the request is rejected at the schema/validation layer, before policy even runs.

---

## E. Credential Lifecycle

### Principles

1. Credentials live in F4 Vault (filesystem boundary).
2. F5 accesses credentials ONLY through F4 Vault MCP interface.
3. F5 does NOT know vault filesystem implementation.
4. F5 does NOT cache credentials beyond a single execution.
5. F5 does NOT write credentials to disk, logs, or error messages.
6. F5 does NOT include credentials in audit entries.

### Resolution Flow

```
CloudExecutor.execute(request)
  │
  ├─ 1. Validate provider_id exists in registry
  ├─ 2. Get auth_vault_id from ProviderConfig (e.g. "cloud/openrouter/api_key")
  ├─ 3. Call vault MCP: read_secret(secret_id=auth_vault_id)
  │     └─ This goes through: Gate → MCP client → Vault MCP server → filesystem
  │     └─ Vault MCP server reads /data/vault/cloud__openrouter__api_key.secret
  │     └─ Returns: {"content": "sk-..."}
  ├─ 4. Extract api_key string from response
  ├─ 5. Use api_key in HTTP Authorization header
  ├─ 6. HTTP call to provider endpoint
  ├─ 7. Response received → api_key reference discarded
  └─ 8. Function returns → api_key no longer referenced
```

### Credential Object Lifetime

```python
class CredentialResolver:
    def resolve(self, provider_id: str) -> str:
        """Resolve API key from vault. Returns key string.
        
        The returned string is used by the caller for a single HTTP request.
        No reference is stored beyond the calling function's scope.
        """
        spec = self.registry.get(provider_id)
        if spec is None:
            raise ProviderNotFoundError(provider_id)
        
        # Call vault MCP through gate's MCP client
        result = self.mcp_client.call("vault", "read_secret", {
            "secret_id": spec.auth_vault_id
        })
        
        if result.status != "ok":
            raise CredentialResolutionError(provider_id)
        
        # Extract content — this is the ONLY time the key exists in F5 code
        api_key = result.data.get("content", "")
        if not api_key:
            raise CredentialEmptyError(provider_id)
        
        return api_key  # caller uses and discards
```

### What F5 Does NOT Do

- ❌ Store api_key in a class attribute
- ❌ Store api_key in a module-level variable
- ❌ Write api_key to any file
- ❌ Include api_key in any log message
- ❌ Include api_key in any exception message
- ❌ Include api_key in audit entries
- ❌ Pass api_key to any subprocess
- ❌ Serialize api_key to JSON
- ❌ Include api_key in the execution workspace

### Vault MCP Boundary

F5 depends on:
- `gate.mcp.client.McpClient.call("vault", "read_secret", {"secret_id": "..."})`

F5 does NOT depend on:
- Vault filesystem path (`/data/vault/`)
- Vault server implementation (`vault_server.py`)
- Vault security module (`vault_security.py`)
- Any vault internal detail

---

## F. Provider Abstraction

### ProviderConfig (from JSON)

```json
{
  "providers": [
    {
      "provider_id": "openrouter",
      "display_name": "OpenRouter",
      "endpoint": "https://openrouter.ai/api/v1",
      "auth_vault_id": "cloud/openrouter/api_key",
      "auth_header": "Authorization",
      "auth_prefix": "Bearer ",
      "timeout_seconds": 60,
      "max_concurrent": 2,
      "priority": 10,
      "models": {
        "anthropic/claude-sonnet-4": {
          "display_name": "Claude Sonnet 4",
          "context_window": 200000,
          "capabilities": ["chat", "tool_use"],
          "max_output_tokens": 8192,
          "pricing": {
            "input_per_1k": 0.003,
            "output_per_1k": 0.015
          }
        },
        "google/gemini-2.5-flash": {
          "display_name": "Gemini 2.5 Flash",
          "context_window": 1048576,
          "capabilities": ["chat", "tool_use"],
          "max_output_tokens": 8192,
          "pricing": {
            "input_per_1k": 0.00015,
            "output_per_1k": 0.0006
          }
        }
      },
      "default_model": "anthropic/claude-sonnet-4"
    }
  ]
}
```

### Key Properties

| Property | Type | Required | Description |
|---|---|---|---|
| `provider_id` | string | yes | Unique identifier (e.g. "openrouter") |
| `display_name` | string | yes | Human-readable name |
| `endpoint` | string | yes | Base URL (HTTPS only) |
| `auth_vault_id` | string | yes | Vault secret_id for API key |
| `auth_header` | string | yes | HTTP header name (default: "Authorization") |
| `auth_prefix` | string | yes | Header prefix (default: "Bearer ") |
| `timeout_seconds` | float | yes | Per-request timeout |
| `max_concurrent` | int | yes | Max concurrent requests to this provider |
| `priority` | int | yes | Fallback order (higher = preferred) |
| `models` | dict | yes | model_id → ModelSpec |
| `default_model` | string | yes | Default model for this provider |

### ModelSpec

| Property | Type | Required | Description |
|---|---|---|---|
| `display_name` | string | yes | Human-readable name |
| `context_window` | int | yes | Max input tokens |
| `capabilities` | list[string] | yes | ["chat", "completion", "tool_use", "embedding"] |
| `max_output_tokens` | int | yes | Max output tokens |
| `pricing` | object | yes | input_per_1k, output_per_1k (USD) |

### Registry Loading

```python
class ProviderRegistry:
    def __init__(self, config_path: str):
        """Load provider definitions from JSON config file."""
        with open(config_path) as f:
            data = json.load(f)
        self._providers = {}
        for p in data.get("providers", []):
            spec = ProviderConfig(**p)
            self._providers[spec.provider_id] = spec
    
    def get(self, provider_id: str) -> ProviderConfig | None:
        return self._providers.get(provider_id)
    
    def all_providers(self) -> list[ProviderConfig]:
        return list(self._providers.values())
    
    def models_for(self, provider_id: str) -> dict[str, ModelSpec]:
        spec = self.get(provider_id)
        return spec.models if spec else {}
```

### Validation Rules

- `endpoint` MUST be HTTPS (reject HTTP)
- `endpoint` MUST NOT contain user info (no `user:pass@host`)
- `endpoint` MUST be a valid URL (no arbitrary strings)
- `auth_vault_id` MUST be a valid vault secret_id (logical format)
- `provider_id` MUST match `^[a-z0-9][a-z0-9._-]*$`
- `model_id` MUST match `^[a-zA-Z0-9._/-]+$`
- Unknown fields in config are rejected (strict parsing)

---

## G. Network Boundary

### Allowed Endpoints

F5 ONLY connects to endpoints defined in the provider registry. No other outbound HTTP is permitted from F5 code.

### TLS Requirement

- All provider endpoints MUST use HTTPS
- TLS certificate verification is MANDATORY (no `verify=False`)
- Uses stdlib `urllib.request` with default SSL context
- No custom certificate bundles
- No proxy configuration

### Request Constraints

| Constraint | Value | Enforcement |
|---|---|---|
| Max request body | 1 MiB | Pre-execution check |
| Max response body | 4 MiB | Streaming read limit |
| Timeout (connect) | 10 seconds | Socket timeout |
| Timeout (read) | 60 seconds (configurable) | Per-provider config |
| Max concurrent | 2 per provider | Semaphore |
| Max concurrent total | 4 | Global semaphore |
| Redirect | FOLLOW max 3 | urllib default |
| DNS | System resolver only | No custom DNS |

### What F5 Does NOT Allow

- ❌ Arbitrary URLs from caller
- ❌ Custom headers from caller
- ❌ HTTP (non-TLS) connections
- ❌ WebSocket connections
- ❌ FTP, SSH, or other protocols
- ❌ SOCKS/HTTP proxy
- ❌ IP address endpoints (must be hostname)
- ❌ Connection to private IP ranges (10.x, 172.16-31.x, 192.168.x, 169.254.x)
- ❌ Connection to localhost/loopback (cloud = external only)

### Endpoint Validation

```python
def validate_endpoint(url: str) -> str:
    """Validate provider endpoint. HTTPS only, no private IPs."""
    from urllib.parse import urlparse
    parsed = urlparse(url)
    
    if parsed.scheme != "https":
        raise ValueError(f"endpoint must be HTTPS: {url}")
    if parsed.hostname is None:
        raise ValueError(f"endpoint must have a hostname: {url}")
    if parsed.username or parsed.password:
        raise ValueError(f"endpoint must not contain user info: {url}")
    
    # Block private IPs (defense-in-depth)
    import ipaddress, socket
    try:
        ip = ipaddress.ip_address(socket.gethostbyname(parsed.hostname))
        if ip.is_private or ip.is_loopback or ip.is_link_local:
            raise ValueError(f"endpoint must not resolve to private IP: {ip}")
    except socket.gaierror:
        raise ValueError(f"endpoint hostname cannot be resolved: {parsed.hostname}")
    
    return url
```

---

## H. Execution Lifecycle

### State Machine

```
PENDING
  ↓ (validated, policy check passed)
APPROVAL_WAIT (if policy requires approval)
  ↓ (approved by admin)
CREDENTIAL_RESOLVE
  ↓ (api_key obtained from vault)
CONNECTING
  ↓ (TCP+TLS connection established)
RUNNING
  ↓ (request sent, response streaming)
COMPLETED | FAILED | TIMEOUT | CANCELLED
```

### States

| State | Description | Auditable |
|---|---|---|
| `PENDING` | Request received, validated | No (pre-policy) |
| `APPROVAL_WAIT` | Waiting for admin approval | Yes |
| `CREDENTIAL_RESOLVE` | Resolving credential from vault | Yes (credential access logged) |
| `CONNECTING` | Establishing HTTP connection | No (ephemeral) |
| `RUNNING` | Request in flight | Yes (periodic heartbeat) |
| `COMPLETED` | Success, response returned | Yes (final audit) |
| `FAILED` | Provider error, malformed response | Yes (final audit) |
| `TIMEOUT` | Connection or read timeout | Yes (final audit) |
| `CANCELLED` | Caller cancelled execution | Yes (final audit) |

### Execution Record

```python
@dataclass
class ExecutionRecord:
    execution_id: str          # unique, generated by F5
    request_id: str            # from caller
    ticket_id: str | None      # from caller (optional)
    provider_id: str           # which provider
    model_id: str              # which model
    requester: str             # who requested (principal.id)
    role: str                  # role of requester
    state: str                 # current state
    created_at: float          # timestamp
    started_at: float | None   # when RUNNING began
    completed_at: float | None # when terminal state reached
    duration_ms: float | None  # total duration
    tokens_input: int | None   # provider-reported input tokens
    tokens_output: int | None  # provider-reported output tokens
    cost_usd: float | None     # estimated cost (from pricing table)
    error_class: str | None    # error type if failed
    error_message: str | None  # error message (redacted — no credentials)
```

### What is Persisted

- `/secondary/logs/executions.jsonl` — append-only execution records (metadata only)
- No prompt/response bodies by default
- No credential material
- No raw HTTP headers

### What is NOT Persisted

- Prompt content (unless explicitly justified for debugging)
- Response body (unless explicitly justified for debugging)
- HTTP headers
- Credential values
- Partial streaming data

---

## I. Runtime Storage / Retention

### Directory Layout

```
/secondary/
    runtime/                    # ephemeral execution workspaces (F5)
        {execution_id}/         # per-execution, cleaned after retention
            metadata.json       # execution record (JSON)
    logs/                       # operational logs
        executions.jsonl        # append-only execution history
        quota.json              # quota state (mutable)
    cache/                      # EXISTING — DO NOT TOUCH
    backups/                    # empty — reserved
```

### Permissions

| Path | Owner | Mode | Purpose |
|---|---|---|---|
| `/secondary/runtime/` | root:root | 0755 | Parent directory |
| `/secondary/runtime/{id}/` | root:root | 0700 | Per-execution workspace |
| `/secondary/logs/` | root:root | 0755 | Log directory |
| `/secondary/logs/executions.jsonl` | root:root | 0644 | Execution history |
| `/secondary/logs/quota.json` | root:root | 0644 | Quota state |

### Retention

| Data | Retention | Cleanup |
|---|---|---|
| Execution workspace | 24 hours (completed), 72 hours (failed) | Cron/systemd timer, daily |
| Executions log | 90 days | Rollover, compress old |
| Quota state | Rolling (reset daily/monthly) | In-place update |

### Crash / Reboot Behavior

- Execution workspaces for running executions are orphaned → cleaned on next boot
- `executions.jsonl` is append-only → no corruption on crash
- `quota.json` may have stale data → acceptable (quota is operational, not security)
- No credential data is ever in /secondary → no credential leak on crash

### Credential Exclusion

F5 code MUST NOT write any file containing credential material to /secondary. This is enforced by:
1. Credential is a local variable in `execute()`
2. No function in the call chain persists credentials
3. Audit redaction catches any accidental inclusion

---

## J. Quota / Resource Accounting

### Separation from Audit

| Concern | Owner | Format | Mutable? |
|---|---|---|---|
| Security/event history | Audit (F2) | JSONL + hash-chain | No (append-only) |
| Operational accounting | Quota (F5) | JSON (mutable) | Yes (counters updated) |

These are COMPLETELY SEPARATE. Quota does not write to audit. Audit does not write to quota.

### Quota Record

```python
@dataclass
class QuotaRecord:
    provider_id: str
    model_id: str
    period: str              # "daily" or "monthly"
    period_start: str        # ISO date "2026-08-17"
    requests_count: int
    tokens_input: int
    tokens_output: int
    estimated_cost_usd: float
    failures_count: int
    last_updated: float      # timestamp
```

### Enforcement

- **Pre-execution:** Check if quota allows the request (count, tokens, cost)
- **Post-execution:** Update counters with actual usage
- **Provider-reported usage:** Use provider's `usage` field in response (most providers include this)
- **Fallback estimation:** If provider doesn't report usage, estimate from pricing table
- **Unknown usage:** Log as `tokens_input=0, tokens_output=0, estimated_cost_usd=0` — do not block

### Limits (Configurable)

```json
{
  "quota": {
    "daily": {
      "max_requests": 100,
      "max_input_tokens": 1000000,
      "max_output_tokens": 500000,
      "max_cost_usd": 10.0
    },
    "monthly": {
      "max_requests": 2000,
      "max_input_tokens": 20000000,
      "max_output_tokens": 10000000,
      "max_cost_usd": 200.0
    }
  }
}
```

### Failure Accounting

Failures are tracked but do NOT count against quota limits. A provider returning 5xx should not exhaust the user's daily request quota.

---

## K. Fallback Rules

### Fallback Chain

Providers are ordered by `priority` (higher = preferred). Fallback attempts the next provider in the chain.

### When Fallback Happens

| Trigger | Fallback? | Reason |
|---|---|---|
| Provider timeout | YES | Provider unreachable |
| Provider 429 (rate limit) | YES | Temporary, try alternative |
| Provider 5xx (server error) | YES | Provider degraded |
| Connection refused | YES | Provider down |
| DNS resolution failure | YES | Provider unreachable |
| TLS error | NO | Possible MITM, do not retry |
| Provider 401 (auth error) | NO | Credential issue, not provider issue |
| Provider 400 (bad request) | NO | Our bug, not provider issue |
| Policy denial | NO | Policy is final |
| Quota exceeded | NO | Quota is final |
| Invalid request schema | NO | Our bug, not provider issue |
| Credential unavailable | NO | Vault issue, not provider issue |

### Fallback Security Rules

1. **Policy applies to EACH fallback attempt.** If provider B requires approval and admin hasn't approved, fallback to B is denied.
2. **Credential is resolved per-attempt.** Different providers may have different vault secret_ids.
3. **Each attempt is audited.** Both the original failure and the fallback attempt are logged.
4. **Quota is checked per-attempt.** If fallback provider has different quota, check it.
5. **No blind retry.** Maximum 1 fallback attempt per execution (no infinite chains).

### Fallback NOT Triggered For

- Policy denial (policy is authoritative)
- Quota exceeded (quota is authoritative)
- Auth failure (credential issue, not provider issue)
- Bad request (our bug)
- Cancellation by caller

---

## L. Streaming Semantics

### Flow

```
Caller → F5:
  POST /cloud/execute { "stream": true, ... }

F5 → Provider:
  POST {provider_endpoint}/v1/chat/completions { "stream": true, ... }

Provider → F5:
  SSE: data: {"choices": [{"delta": {"content": "Hello"}}]}
  SSE: data: {"choices": [{"delta": {"content": " world"}}]}
  SSE: data: [DONE]

F5 → Caller:
  SSE: event: uta_stream
       data: {"execution_id": "...", "chunk": "Hello"}
  SSE: event: uta_stream
       data: {"execution_id": "...", "chunk": " world"}
  SSE: event: uta_stream
       data: {"execution_id": "...", "status": "completed", "usage": {...}}
```

### Streaming Rules

1. **No buffering entire response.** F5 relays chunks as they arrive.
2. **Timeout per chunk.** If no chunk arrives for 30 seconds, connection is closed, execution marked TIMEOUT.
3. **Cancellation propagation.** If caller disconnects, F5 closes provider connection.
4. **Partial response on failure.** If stream disconnects mid-way, partial chunks already sent remain valid. Final event reports partial status.
5. **Credential never in stream.** Only the Authorization header contains the credential; it is not echoed in SSE events.
6. **Audit on completion.** Final audit entry records total tokens, duration, status — not individual chunks.

### Non-Streaming

For non-streaming requests, F5 waits for complete response, then returns it as JSON. Same timeout applies (60 seconds default).

---

## M. MCP Cloud Tool Contracts

### Tool: `cloud.execute`

**Purpose:** Execute a cloud AI inference request.

**Input Schema:**
```json
{
  "type": "object",
  "properties": {
    "provider_id": {"type": "string"},
    "model_id": {"type": "string"},
    "messages": {"type": "array", "items": {"type": "object"}},
    "max_tokens": {"type": "integer", "default": 4096},
    "temperature": {"type": "number", "default": 0.7},
    "stream": {"type": "boolean", "default": false},
    "ticket_id": {"type": "string"}
  },
  "required": ["provider_id", "model_id", "messages"]
}
```

**Output Schema:**
```json
{
  "status": "ok",
  "data": {
    "execution_id": "...",
    "provider_id": "...",
    "model_id": "...",
    "content": "...",
    "usage": {"input_tokens": 100, "output_tokens": 50},
    "cost_usd": 0.001,
    "duration_ms": 1234
  }
}
```

**Policy capability:** `NETWORK`
**Auth required:** Yes (token)
**Approval:** Per policy rule (provider-specific)
**Sensitive fields:** `content` (may contain AI-generated text, not credentials)

### Tool: `cloud.list_providers`

**Purpose:** List configured cloud providers and their models.

**Input Schema:**
```json
{
  "type": "object",
  "properties": {}
}
```

**Output Schema:**
```json
{
  "status": "ok",
  "data": {
    "providers": [
      {
        "provider_id": "openrouter",
        "display_name": "OpenRouter",
        "models": ["anthropic/claude-sonnet-4", "google/gemini-2.5-flash"],
        "default_model": "anthropic/claude-sonnet-4"
      }
    ]
  }
}
```

**Policy capability:** `READ`
**Auth required:** Yes
**Approval:** No
**Sensitive fields:** None (no credentials exposed)

### Tool: `cloud.execution_status`

**Purpose:** Check status of a cloud execution.

**Input Schema:**
```json
{
  "type": "object",
  "properties": {
    "execution_id": {"type": "string"}
  },
  "required": ["execution_id"]
}
```

**Output Schema:**
```json
{
  "status": "ok",
  "data": {
    "execution_id": "...",
    "state": "COMPLETED",
    "provider_id": "...",
    "model_id": "...",
    "duration_ms": 1234,
    "tokens_input": 100,
    "tokens_output": 50,
    "cost_usd": 0.001
  }
}
```

**Policy capability:** `READ`
**Auth required:** Yes
**Approval:** No
**Sensitive fields:** None

### Tool: `cloud.quota_status`

**Purpose:** Check quota usage for cloud providers.

**Input Schema:**
```json
{
  "type": "object",
  "properties": {
    "provider_id": {"type": "string"},
    "period": {"type": "string", "enum": ["daily", "monthly"]}
  }
}
```

**Output Schema:**
```json
{
  "status": "ok",
  "data": {
    "provider_id": "openrouter",
    "period": "daily",
    "requests_count": 15,
    "max_requests": 100,
    "tokens_input": 50000,
    "tokens_output": 25000,
    "estimated_cost_usd": 0.45,
    "max_cost_usd": 10.0
  }
}
```

**Policy capability:** `READ`
**Auth required:** Yes
**Approval:** No
**Sensitive fields:** None

### Tool: `cloud.cancel`

**Purpose:** Cancel a running cloud execution.

**Input Schema:**
```json
{
  "type": "object",
  "properties": {
    "execution_id": {"type": "string"}
  },
  "required": ["execution_id"]
}
```

**Output Schema:**
```json
{
  "status": "ok",
  "data": {
    "execution_id": "...",
    "state": "CANCELLED"
  }
}
```

**Policy capability:** `EXECUTE`
**Auth required:** Yes
**Approval:** No (caller can cancel their own; admin can cancel any)
**Sensitive fields:** None

---

## N. F2 / F3 / F4 Integration Contracts

### Dependency Diagram

```
F2 (Gate Core)
  ├─ AuthProvider         ← F5 uses for token validation
  ├─ PolicyEngine         ← F5 uses for authorization
  ├─ ApprovalStore        ← F5 uses for approval flow
  ├─ AuditLog             ← F5 uses for audit entries
  └─ EventBus             ← F5 uses for SSE events

F3 (MCP Boundary)
  ├─ McpClient            ← F5 uses for vault credential resolution
  ├─ Catalog              ← F5 adds cloud tools to catalog
  └─ MCP Servers          ← F5 adds cloud_server

F4 (Vault)
  ├─ Vault MCP Server     ← F5 reads credentials through this
  └─ Vault Security       ← F5 does NOT depend on this directly
```

### F2 Contract

F5 adds a new method to `Gate` class: `handle_cloud_execution(principal, body)`. This follows the same pattern as `handle_mcp()`:

1. Validate request schema
2. Validate provider/model exist in registry
3. Policy evaluate (role + capability + resource)
4. If approval required → approval flow
5. Execute cloud call
6. Audit
7. Return result

F5 does NOT modify `handle_command()` or `handle_mcp()`.

### F3 Contract

F5 adds new tools to the MCP catalog:

```python
# cloud (F5 — cloud execution boundary)
_register("cloud", "execute",          Capability.NETWORK.value, ScopeKind.SYSTEM, "execute cloud AI inference")
_register("cloud", "list_providers",   Capability.READ.value,    ScopeKind.SYSTEM, "list configured providers")
_register("cloud", "execution_status", Capability.READ.value,    ScopeKind.SYSTEM, "check execution status")
_register("cloud", "quota_status",     Capability.READ.value,    ScopeKind.SYSTEM, "check quota usage")
_register("cloud", "cancel",           Capability.EXECUTE.value, ScopeKind.SYSTEM, "cancel running execution")
```

Note: `ScopeKind.SYSTEM` is reused for cloud tools because the resource is "the cloud provider" (not a filesystem path, not a vault secret). This is a logical resource scope.

### F4 Contract

F5 calls vault through MCP client:

```python
result = self.mcp_client.call("vault", "read_secret", {
    "secret_id": "cloud/openrouter/api_key"
})
```

F5 does NOT:
- Import vault_server.py
- Import vault_security.py
- Access /data/vault/ directly
- Know vault file naming convention
- Know vault permission model

---

## O. Failure / Recovery Matrix

| Failure | Behavior | Credential Safe? | Audit? | Recovery |
|---|---|---|---|---|
| Provider timeout | Mark TIMEOUT, try fallback | YES | YES | Retry with different provider |
| Provider 429 | Mark FAILED, try fallback | YES | YES | Retry after delay or with different provider |
| Provider 5xx | Mark FAILED, try fallback | YES | YES | Retry with different provider |
| Provider auth failure (401) | Mark FAILED, NO fallback | YES | YES | Check vault credential |
| Malformed response | Mark FAILED, NO fallback | YES | YES | Check provider config |
| Stream disconnect | Mark FAILED (partial), NO fallback | YES | YES | Retry request |
| Vault unavailable | Mark FAILED, NO fallback | YES (not resolved) | YES | Check vault service |
| Quota exceeded | Mark FAILED, NO fallback | YES | YES | Wait for quota reset |
| Cancellation | Mark CANCELLED | YES | YES | N/A |
| Process crash | Orphaned workspace, cleaned on reboot | YES (credential never on disk) | Partial (pre-crash entries intact) | systemd restart |
| Reboot during execution | Orphaned workspace, cleaned on boot | YES (credential never on disk) | Partial | systemd restart |

### Crash Safety

- Credential is a Python string variable in `execute()` scope
- On process crash, Python process memory is released by OS
- No credential data is persisted to disk by F5
- Audit entries before crash are intact (append-only, fsync'd)
- Orphaned runtime workspaces contain only metadata.json (no credentials)

---

## P. Explicit Non-Goals

F5 DOES NOT include:

1. **Telegram UX** — belongs to later phase (L1 channel)
2. **Heartbeat / scheduled execution** — belongs to F6
3. **Autonomous agent loops** — F5 is a runtime, not an agent
4. **LLM router** — F5 uses deterministic config, not LLM-based routing
5. **Local model replacement** — ai-gateway/llama-server remain separate
6. **14B/7B experiments** — belongs to F7 (optional)
7. **OpenCode replacement** — F5 is a component, not a product
8. **Generic shell execution** — F2 runner handles that
9. **Arbitrary HTTP proxy** — F5 only calls registered providers
10. **Production credential migration** — separate approval-gated task
11. **Multi-user quota management** — single-user design for now
12. **Provider failover across regions** — single-endpoint per provider
13. **Cost optimization / model selection** — deterministic config, not AI-driven
14. **Prompt management / template system** — caller provides messages directly
15. **Response caching** — no caching layer in F5
16. **Rate limiting beyond provider limits** — provider's 429 is the rate limit

---

## Q. Open Questions Requiring Human Decision

1. **Should `cloud.execute` be accessible to the `user` role?** Currently the design allows it via policy rules. If only admin should use cloud execution, the policy rules can restrict it. Recommendation: allow both admin and user, with provider-specific policy rules.

2. **Should F5 add a new HTTP endpoint (`/cloud/execute`) or only MCP tools?** MCP tools are the standard interface. Adding a separate HTTP endpoint would be redundant. Recommendation: MCP tools only.

3. **Should response bodies be persisted for debugging?** Currently metadata-only. If debugging is needed, response bodies could be optionally persisted. Recommendation: metadata-only, add opt-in later if needed.

4. **Should F5 support non-chat completions (embeddings, etc.)?** The provider abstraction supports it via `capabilities` field. Recommendation: support chat only in initial implementation, extend later.

5. **Should the providers.json config be in the gate repo or in /etc?** Gate repo is simpler for development. /etc is more production-standard. Recommendation: gate repo for now, move to /etc in F8.

6. **Should F5 create a systemd service or run inside the existing gate process?** F5 is a module of the gate, not a separate service. It runs inside the gate process. Recommendation: module, not service.

---

## R. Implementation Plan

### Step 1: Provider Abstraction + Registry
- `gate/cloud/provider.py` — dataclasses
- `gate/cloud/registry.py` — JSON loader, validation
- `gate/cloud/config.py` — CloudConfig
- `providers.json` — sample config with 1 provider
- Tests: ~10

### Step 2: Credential Resolver
- `gate/cloud/credential.py` — CredentialResolver (vault MCP)
- Tests: ~8 (mock vault MCP)

### Step 3: Execution Lifecycle
- `gate/cloud/lifecycle.py` — state machine, ExecutionRecord
- `gate/cloud/errors.py` — F5 errors
- Tests: ~8

### Step 4: Cloud Executor
- `gate/cloud/executor.py` — HTTP call, response parsing
- `gate/cloud/streaming.py` — SSE proxy
- Tests: ~12 (mock HTTP)

### Step 5: Quota Tracker
- `gate/cloud/quota.py` — QuotaTracker, persistence
- Tests: ~8

### Step 6: MCP Integration
- `gate/mcp/servers/cloud_server.py` — 5 MCP tools
- Update `gate/mcp/catalog.py` — add cloud tools
- Update `gate/policy/engine.py` — add cloud rules
- Tests: ~10

### Step 7: Gate Core Integration
- Update `gate/core.py` — `handle_cloud_execution()`
- Update `gate/factory.py` — wire CloudConfig
- Tests: ~8

### Step 8: Documentation + Reports
- `F5-REPORT.md`
- Update `UTA-PLAN.md`

**Total estimated: ~64 new tests**

---

## S. Verification / Test Plan

### Unit Tests

- Provider config validation (valid/invalid endpoints, missing fields)
- Registry loading (valid JSON, missing file, malformed JSON)
- Credential resolution (mock vault MCP, failure paths)
- Lifecycle state machine (all transitions)
- Quota tracking (increment, check, reset)
- Endpoint validation (HTTPS only, no private IPs)
- Streaming relay (mock provider SSE)

### Integration Tests

- Full execution flow (mock provider HTTP)
- Policy enforcement (allow/deny per provider)
- Approval flow for cloud execution
- Fallback chain (provider failure → fallback)
- Audit entry verification (all fields present, no secrets)
- Quota enforcement (limit reached → denied)

### Security Tests

- Credential not in audit (verify audit content)
- Credential not in error messages (verify exception strings)
- Credential not in /secondary (verify no file contains api_key)
- Arbitrary URL rejection (verify non-registry URLs are blocked)
- HTTP rejection (verify HTTP endpoints are rejected)
- Private IP rejection (verify 10.x, 192.168.x are blocked)
- TLS verification (verify certificate is checked)

### Regression Tests

- All 148 F2/F3/F4 tests still pass
- No existing behavior changed

---

## T. Rollback Plan

F5 is additive — new module, new catalog entries, new policy rules. Rollback:

1. Remove `gate/cloud/` directory
2. Remove cloud tools from `gate/mcp/catalog.py`
3. Remove cloud rules from `gate/policy/engine.py`
4. Remove `providers.json`
5. All 148 existing tests pass (F2/F3/F4 unchanged)

No production services are modified. No credentials are moved. No system changes.

Rollback is safe and instant.

---

```
F5 DESIGN REVIEW: COMPLETE
IMPLEMENTATION: NOT STARTED
SYSTEM CHANGES: NONE
INSTALLATIONS: NONE
PRODUCTION SERVICES: UNCHANGED
```
