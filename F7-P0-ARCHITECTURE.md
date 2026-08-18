# F7-P0 — Local Intelligence Runtime: Architecture & Preflight Report

> Status: CORRECTED PREFLIGHT / DESIGN
> Date: 2026-08-18
> Baseline: F6 CONFIRMED CLOSED (304 tests, commit `04dbb59`)
> Previous: F7-P0 original (superseded by this correction)

---

## 1. Current Repository State

### Source Files
- 56 Python files in `gate/gate/`
- 12 test files in `gate/tests/`
- 304 tests passing, 0 regression
- Modules: approval, audit, auth, cloud (13 files), config, core, errors, events, factory, health, heartbeat (5 files), ingress, mcp (9 files), policy, runner

### Key Files for F7 Integration
| File | Purpose | F7 Relevance |
|------|---------|--------------|
| `core.py` | Gate orchestrator: auth → schema → policy → approval → execute → audit | **Primary integration point** |
| `policy/engine.py` | Deny-by-default policy evaluation | Model requests pass through here |
| `policy/classes.py` | Capability enum: READ, WRITE, EXECUTE, SECRET, NETWORK, BACKGROUND | See Section 13 for AGENT decision |
| `mcp/catalog.py` | Tool catalog: ToolSpec(server_id, tool_id, capability, scope_kind) | Model queries this for available tools |
| `mcp/client.py` | MCP tool execution with timeout, result limits | Agent loop calls this for tool results |
| `ingress/schemas.py` | Request validation: CommandRequest, McpRequest, ApproveRequest | Source list needs "agent" added |
| `events.py` | EventBus for SSE streaming | Agent loop publishes trajectory events |
| `audit/audit.py` | Hash-chain append-only audit log | Agent actions logged here |
| `approval/store.py` | Approval state machine | Write operations require approval |
| `factory.py` | build_gate() — single construction point | F7 provider wiring goes here |
| `config.py` | Config dataclass, env loading | F7 config fields added here |

---

## 2. F6 Baseline Verification

F6 is the authoritative baseline for F7. Verified:

- **304/304 tests PASS**, 0 regression
- Commits: `686f8b1`, `d388ff5`
- systemd timer active: `uta-heartbeat.timer` (5-min interval)
- All 4 heartbeat tasks verified working
- `source=background` audit entries confirmed in audit log
- Hash chain integrity: OK
- No secrets in audit log: VERIFIED
- F2/F3/F4/F5A security boundaries: ALL INTACT

### F6 Source/Context Pattern (Authoritative for F7)

F6 establishes the distinction between **source/context** and **capability**:

| Aspect | F6 Example | Role |
|--------|-----------|------|
| **Source** | `"background"` | Execution context — WHO/WHAT initiated the action |
| **Capability** | `BACKGROUND` | Authorization token — WHAT the policy evaluates |
| **Command** | `"background.health_check"` | Specific operation name |
| **Role** | `"admin"` | Policy identity — WHO is requesting |

Key invariants from F6:
- **"Heartbeat privilege = interactive, never more"**
- **"Heartbeat does not bypass policy/approval"**
- BACKGROUND capability is granted only to admin role on specific hardcoded tasks
- User role gets NO BACKGROUND rules → denied by default
- Audit entries record source=background for traceability

**F7 MUST follow this same pattern.** The agent is a source/context. It does not gain authority from being an agent.

---

## 3. F2–F6 Integration Points

### 3.1 Gate Core Pipeline (F2)

```
handle_mcp(principal, body):
  1. validate_mcp_request(body)     → McpRequest
  2. tool_lookup(server_id, tool_id) → ToolSpec
  3. _mcp_validate_scope(req, spec)  → resource
  4. policy.evaluate(role, cmd, cap, resource) → Decision
  5. if needs_approval → _mcp_approval_flow()
  6. mcp_client.call(server_id, tool_id, args) → McpResult
  7. audit.append(...)
  8. return result
```

**F7 integration**: Agent tool calls enter at step 1. F7 wraps model output into McpRequest-compatible format and feeds through existing pipeline. Agent does NOT bypass any step.

### 3.2 Policy Engine (F2)

```python
Decision = policy.evaluate(
    role="admin",
    command="filesystem.read_file",
    capability="READ",
    resource="/data/some/path",
)
```

**F7 integration**: Model tool calls use the invoking principal's role (e.g. role=principal.role). Policy decides based on (role, tool's capability, command). Model identity does not influence the decision.

### 3.3 MCP Catalog (F3)

```python
CATALOG = {
    ("filesystem", "read_file"): ToolSpec(
        server_id="filesystem", tool_id="read_file",
        capability="READ", scope_kind=ScopeKind.PATH,
    ),
    ...
}
```

**F7 integration**: Agent runtime builds model's tool schema from CATALOG. Each tool carries its own capability. Model sees available tools but F2 determines if each is permitted.

### 3.4 MCP Client (F3)

```python
result = mcp_client.call("filesystem", "read_file", {"path": "/data/file.txt"})
# McpResult(status="ok", data={...}, duration_ms=123.4)
```

**F7 integration**: Agent loop calls this after policy allows. Results fed back to model as tool results in trajectory.

### 3.5 Audit (F2)

```python
audit.append({
    "source": "agent",        # execution context
    "request_id": "req_...",
    "command": "filesystem.read_file",
    "capability": "READ",     # tool's capability, not agent capability
    "decision": "allowed",
    ...
})
```

**F7 integration**: Agent loop logs every tool call. Source="agent" for traceability. Tool's own capability recorded.

### 3.6 EventBus (F2)

```python
events.publish({"type": "agent_tool_call", "request_id": "...", "tool": "filesystem.read_file"})
```

**F7 integration**: Agent publishes trajectory events for SSE streaming.

### 3.7 Approval Gate (F2)

```python
if decision.needs_approval:
    return approval_flow(...)
```

**F7 integration**: Model write requests go through approval. Agent loop pauses, waits for user confirmation. Model cannot skip approval.

---

## 4. Existing Reusable Abstractions

| Abstraction | Location | F7 Usage |
|-------------|----------|----------|
| `Capability` enum | `policy/classes.py` | **No new capability needed** (see Section 13) |
| `ToolSpec` | `mcp/catalog.py` | Model tool schema source — each tool has its own capability |
| `Decision` | `policy/engine.py` | Tool-call authorization |
| `McpResult` | `mcp/client.py` | Tool execution result |
| `McpRequest` | `ingress/schemas.py` | Tool-call request format |
| `EventBus` | `events.py` | Agent trajectory events |
| `AuditLog` | `audit/audit.py` | Agent action audit |
| `ApprovalStore` | `approval/store.py` | Write-operation approval |
| `build_mcp_rules()` | `policy/engine.py` | Policy rules for MCP tools |
| `_SOURCES` | `ingress/schemas.py` | Needs "agent" added (like F6 added "background") |

---

## 5. Proposed F7 Architecture

### 5.1 New Module Structure

```
gate/gate/
├── agent/                    # NEW — F7
│   ├── __init__.py
│   ├── provider.py           # ModelProvider protocol, ModelRequest, ModelResponse
│   ├── providers/
│   │   ├── __init__.py
│   │   ├── fake.py           # FakeModelProvider (for testing)
│   │   └── local.py          # LocalModelProvider (llama-cpp/ONNX wrapper)
│   ├── session.py            # Session, Trajectory, AgentState
│   ├── loop.py               # AgentLoop — multi-turn inference
│   ├── tool_call.py          # Tool-call parsing, validation, schema generation
│   └── config.py             # AgentConfig (model path, max iterations, etc.)
```

### 5.2 Data Flow

```
User message
    ↓
AgentLoop.run(user_message)
    ↓
Session: append user message
    ↓
Loop (max_iterations):
    ↓
    ModelRequest(messages, tools)
        ↓
    ModelProvider.complete(request)
        ↓
    ModelResponse
        ├── text? → return to user (done)
        └── tool_calls? → for each:
            ↓
            Tool-call parser (extract structured call)
                ↓
            Schema validation (arguments match tool schema)
                ↓
            Gate.handle_agent_tool_call(server_id, tool_id, args, request_id)
                ↓
                Catalog lookup → ToolSpec
                    ↓
                Scope validation
                    ↓
                PolicyEngine.evaluate(role, command, capability, resource)
                    ↓
                Decision?
                    ├── denied → error to model
                    ├── needs_approval → pause, wait for user
                    └── allowed →
                        ↓
                    McpClient.call(server_id, tool_id, args)
                        ↓
                    AuditLog.append(source="agent", ...)
                        ↓
                    McpResult
                ↓
            Sanitize result (strip secrets)
                ↓
            Session: append tool result
                ↓
    Back to model (next iteration)
```

### 5.3 Gate Integration -- No Parallel Authorization

The agent loop does NOT bypass Gate. Two new internal methods on Gate:

```python
def handle_agent_tool_call(
    self,
    *,
    principal: Principal,
    server_id: str,
    tool_id: str,
    args: dict,
    request_id: str,
) -> McpResult:
    # 1. Catalog lookup
    spec = tool_lookup(server_id, tool_id)
    if spec is None:
        raise ValidationError(f"unknown tool: {server_id}.{tool_id}")

    # 2. Scope validation
    resource = self._mcp_validate_scope_for_agent(server_id, tool_id, args, spec)

    # 3. Policy evaluation
    decision = self.policy.evaluate(
        role=principal.role,
        command=spec.command_id,
        capability=spec.capability,
        resource=resource,
    )

    # 4. Denied
    if not decision.allowed:
        self.audit_log.append({
            "source": "agent", "request_id": request_id,
            "command": spec.command_id, "capability": spec.capability,
            "principal_id": principal.id, "result": "denied",
        })
        return McpResult(status="error", error_class="denied",
                         data={"error": f"policy denied: {decision.reason}"})

    # 5. Approval required -- use existing F2 ApprovalStore
    if decision.needs_approval:
        approval = self.approval_store.create(
            request_id=request_id,
            command=spec.command_id,
            capability=spec.capability,
            args=args,
            requester=principal.id,
        )
        self.audit_log.append({
            "source": "agent", "request_id": request_id,
            "command": spec.command_id, "capability": spec.capability,
            "principal_id": principal.id, "result": "approval_requested",
            "approval_id": approval.id,
        })
        return McpResult(status="approval_required",
                         data={"approval_id": approval.id,
                               "request_id": request_id})

    # 6. Execute via MCP
    try:
        result = self.mcp_client.call(server_id, tool_id, args)
    except Exception as e:
        self.audit_log.append({
            "source": "agent", "request_id": request_id,
            "command": spec.command_id, "capability": spec.capability,
            "principal_id": principal.id, "result": "execution_error",
        })
        return McpResult(status="error", error_class="mcp_error",
                         data={"error": str(e)})

    # 7. Audit success
    self.audit_log.append({
        "source": "agent", "request_id": request_id,
        "command": spec.command_id, "capability": spec.capability,
        "principal_id": principal.id, "result": "executed",
    })
    return result


def execute_approved_tool_call(
    self,
    *,
    principal: Principal,
    request_id: str,
) -> McpResult:
    approval = self.approval_store.get(request_id)
    if approval is None or approval.state != ApprovalState.CONSUMED:
        raise ValidationError(f"no consumed approval for request_id={request_id}")

    spec = tool_lookup_from_command(approval.command)
    if spec is None:
        raise ValidationError(f"unknown tool for command: {approval.command}")

    # Re-validate scope
    resource = self._mcp_validate_scope_for_agent(
        spec.server_id, spec.tool_id, approval.arg_hash, spec)

    # Re-evaluate policy (rules may have changed while agent was suspended)
    decision = self.policy.evaluate(
        role=principal.role,
        command=approval.command,
        capability=approval.capability,
        resource=resource,
    )
    if not decision.allowed:
        self.audit_log.append({
            "source": "agent", "request_id": request_id,
            "result": "denied_on_resume",
        })
        return McpResult(status="error", error_class="denied",
                         data={"error": f"policy denied on resume: {decision.reason}"})

    # Execute via MCP (using stored args from approval)
    try:
        result = self.mcp_client.call(spec.server_id, spec.tool_id, approval.args)
    except Exception as e:
        self.audit_log.append({
            "source": "agent", "request_id": request_id,
            "result": "execution_error",
        })
        return McpResult(status="error", error_class="mcp_error",
                         data={"error": str(e)})

    self.audit_log.append({
        "source": "agent", "request_id": request_id,
        "result": "executed",
    })
    return result
```

**Critical**: `capability=spec.capability` -- the capability comes from the ToolSpec in CATALOG, NOT from the agent. The agent is never the source of authority.


---

## 6. ModelProvider Design

### 6.1 Protocol

```python
class ModelProvider(Protocol):
    """Model provider interface. Transport-agnostic."""

    def complete(self, request: ModelRequest) -> ModelResponse:
        """Run inference. Returns structured response."""
        ...

    def is_available(self) -> bool:
        """Check if provider is ready."""
        ...

    def metadata(self) -> ModelMetadata:
        """Provider and model metadata."""
        ...
```

### 6.2 ModelMetadata

```python
@dataclass(frozen=True)
class ModelMetadata:
    provider_id: str           # "local-llama", "cloud-openai", etc.
    model_id: str              # "cactus-needle-0.3b", "gpt-4o", etc.
    display_name: str = ""
    context_window: int = 4096
    max_output_tokens: int = 1024
    supports_tools: bool = True
    supports_streaming: bool = False
    ram_mb: int = 0            # local model RAM usage
    latency_ms_p50: float = 0  # typical inference latency
```

### 6.3 Provider Selection

```python
class ProviderRegistry:
    """Registry of available model providers. Similar to F5 cloud registry."""

    def __init__(self):
        self._providers: dict[str, ModelProvider] = {}

    def register(self, provider_id: str, provider: ModelProvider) -> None: ...
    def get(self, provider_id: str) -> ModelProvider: ...
    def list_available(self) -> list[ModelMetadata]: ...
    def select_best(self, *, requires_tools: bool = True) -> ModelProvider | None:
        """Select best available provider. No AI decision — deterministic."""
        ...
```

---

## 7. ModelRequest/ModelResponse Design

### 7.1 ModelRequest

```python
@dataclass
class ModelRequest:
    messages: list[dict]           # conversation history [{role, content}]
    tools: list[ToolDefinition]    # available tool schemas
    max_tokens: int = 1024
    temperature: float = 0.3       # low temp for deterministic tool calls
    stop: list[str] | None = None
    request_id: str = ""           # for audit tracing
```

### 7.2 ModelResponse

```python
@dataclass
class ModelResponse:
    content: str | None            # text response (if any)
    tool_calls: list[ToolCall]     # tool calls requested (if any)
    finish_reason: str             # "stop", "tool_call", "max_tokens", "error"
    usage: TokenUsage | None = None
    raw: dict | None = None        # provider-specific raw output (not exposed)

@dataclass
class TokenUsage:
    input_tokens: int = 0
    output_tokens: int = 0
```

### 7.3 ToolCall

```python
@dataclass
class ToolCall:
    id: str                        # unique call ID (for result matching)
    server_id: str                 # MCP server (e.g. "filesystem")
    tool_id: str                   # tool name (e.g. "read_file")
    arguments: dict                # parsed arguments
```

### 7.4 ToolDefinition (exposed to model)

```python
@dataclass
class ToolDefinition:
    server_id: str
    tool_id: str
    name: str                      # display name for model prompt
    description: str               # what the tool does
    parameters: dict               # JSON Schema for arguments
    # NOTE: capability is NOT exposed to model — model doesn't need to know
```

---

## 8. Session/Trajectory Design

### 8.1 Session

```python
class Session:
    """Mutable session state. Holds conversation and tool call history."""

    def __init__(self, session_id: str):
        self.session_id = session_id
        self.messages: list[dict] = []        # [{role, content}]
        self.tool_calls: list[ToolCall] = []  # all tool calls made
        self.tool_results: list[dict] = []    # [{tool_call_id, result}]
        self.metadata: dict = {}              # session metadata (NO SECRETS)

    def add_user_message(self, content: str) -> None: ...
    def add_assistant_message(self, content: str) -> None: ...
    def add_tool_call(self, call: ToolCall) -> None: ...
    def add_tool_result(self, tool_call_id: str, result: dict) -> None: ...
    def to_messages(self) -> list[dict]:
        """Build message list for ModelRequest."""
        ...
```

### 8.2 Trajectory

```python
@dataclass
class TrajectoryEntry:
    timestamp: float
    event_type: str              # "user_message", "model_response", "tool_call", "tool_result"
    data: dict                   # event data
    request_id: str = ""         # audit correlation

class Trajectory:
    """Append-only log of agent actions. Read-only view."""

    def __init__(self, session: Session):
        self._session = session
        self._entries: list[TrajectoryEntry] = []

    def append(self, entry: TrajectoryEntry) -> None: ...
    def entries(self) -> list[TrajectoryEntry]: ...
    def to_audit_dict(self) -> dict:
        """Export for audit logging. Sanitized — no secrets."""
        ...
```

### 8.3 Secret Safety

**NEVER stored in session/trajectory:**
- API keys
- Vault credentials
- Raw secrets
- Authentication tokens
- Password material

**Tool results sanitized before storing:**
- Vault read results → content redacted (only secret_id recorded, not value)
- File read results → no credential patterns
- Error messages → no credential leakage

**Audit integration:**
- Trajectory does NOT replace audit log
- Each tool call creates an audit entry via existing `AuditLog.append()`
- Trajectory is for model context; audit is for security record

---

## 9. Tool-Call Design

### 9.1 Parsing

Model output is parsed into structured tool calls:

```python
def parse_tool_calls(response: ModelResponse) -> list[ToolCall]:
    """Parse model output into ToolCall objects.

    Handles:
    - JSON function calling format (OpenAI-style)
    - XML-style tool calls
    - Plain text with tool call markers
    Returns empty list if no tool calls found.
    """
    ...
```

### 9.2 Validation Pipeline

```
Raw model output
    ↓
parse_tool_calls() → list[ToolCall]
    ↓
For each ToolCall:
    ↓
    1. Schema validation (arguments match JSON Schema)
       → reject if invalid
    ↓
    2. Catalog lookup (tool exists in CATALOG?)
       → reject if unknown tool
    ↓
    3. ToolSpec extracted (capability, scope_kind from CATALOG)
       → capability comes from ToolSpec, NOT from model
    ↓
    4. Gate.handle_agent_tool_call(server_id, tool_id, args)
       → policy.evaluate(role, command, ToolSpec.capability, resource)
       → Decision
    ↓
    5. If denied → return error to model
    ↓
    6. If needs_approval → return approval signal to agent loop
    ↓
    7. If allowed → McpClient.call() → McpResult
    ↓
    8. Sanitize McpResult (strip secrets)
    ↓
    9. Append to session
    ↓
    10. Audit entry (source="agent", capability=ToolSpec.capability)
```

### 9.3 Tool Schema Generation

Build tool definitions from CATALOG for the model:

```python
def build_tool_definitions(
    catalog: dict,
    *,
    include_capabilities: frozenset[str] | None = None,
    exclude_servers: frozenset[str] | None = None,
) -> list[ToolDefinition]:
    """Build tool schemas for model from MCP catalog.

    Args:
        include_capabilities: if set, only include tools with these capabilities
        exclude_servers: if set, exclude tools from these servers

    Returns list of ToolDefinition for ModelRequest.tools.
    """
    ...
```

**Tool visibility ≠ permission.** Even if a tool schema is shown to the model, F2 policy decides whether the model can use it. The model may see `vault.write_secret` in its tool list but F2 will deny it (no agent policy rules for vault writes).

---

## 10. Tool-Schema Exposure Design

### What Tools to Show the Model

The model receives tool definitions from CATALOG, filtered by:

- **Include**: Tools with capabilities the agent is expected to use (READ, EXECUTE for common tasks)
- **Exclude**: Tools the agent should never request (SECRET capability, specific vault operations)

### Filtering Strategy

```python
# Default agent tool visibility
AGENT_TOOL_FILTER = {
    "include_capabilities": {Capability.READ.value, Capability.EXECUTE.value},
    "exclude_servers": set(),  # no servers excluded by default
}
```

### Why Show Tools That Will Be Denied?

Showing `vault.write_secret` to the model but having F2 deny it is **correct behavior**:
1. Model learns what tools exist (better context)
2. F2 still enforces policy (model can't bypass)
3. If model tries → denied → audit logged → model learns to not try again
4. This is the same pattern as a human user seeing a tool but not having permission

### Capability is NOT Exposed

ToolDefinition does NOT include the capability field. The model doesn't know that `filesystem.read_file` has capability="READ". It only sees:
- Tool name
- Description
- Parameter schema

This prevents the model from reasoning about capabilities (which is a security concern).

---

## 11. Security Boundary Analysis

### Trust Model

```
UNTRUSTED: Model Output
    ↓ validation
STRUCTURED: Tool Call (parsed, schema-validated)
    ↓ catalog lookup
IDENTIFIED: ToolSpec (capability from CATALOG, not model)
    ↓ policy
AUTHORIZED: Decision (F2 is authority)
    ↓ approval (if needed)
APPROVED: User Confirmation
    ↓ MCP
EXECUTED: Tool Result
    ↓ sanitization
SAFE: Sanitized Result → Model
```

### What Model CAN Do
- Select a tool from visible tools
- Generate arguments for the selected tool
- Request tool execution via structured output
- Receive tool results in context
- Retry with different arguments after failure

### What Model CANNOT Do
- Bypass policy engine (F2 is authority)
- Skip approval gate (F2 is authority)
- Access credentials (F4 is authority)
- Execute arbitrary commands (F3 MCP boundary)
- Modify security policy
- Write to vault without approval
- Access files outside workspace roots
- Control execution priority or privilege
- Create infinite loops (max_iterations enforced)
- Escape audit logging
- Gain authority from being "the agent"

### Threat Model

| Threat | Mitigation |
|--------|-----------|
| Model hallucinates tool call | Schema validation rejects malformed calls |
| Model generates invalid JSON | Parser returns empty list, loop continues |
| Model requests unauthorized tool | F2 policy denies, audit logged |
| Model requests write without approval | Approval gate blocks, user confirms |
| Model tries to access vault/credentials | F2 policy denies (no agent vault rules) |
| Model generates infinite tool calls | max_iterations stops loop |
| Model output contains injection | Tool results sanitized before model sees them |
| Provider fails | Loop handles error, returns to user |
| Tool execution fails | Error captured, loop continues or terminates |
| Model tries to escalate via identity | Agent role=principal.role, same as interactive user, no extra privilege |
| Model tries to use agent source for privilege | Source="agent" is audit-only, policy uses tool capability |

---

## 12. Source/Context vs Capability Analysis

### F6 Precedent

F6 established:
- `source = "background"` → audit context, NOT a capability grant
- `capability = BACKGROUND` → policy evaluates this
- `role = principal.role` → who is requesting (the invoking user)
- Policy: principal.role + BACKGROUND on specific tasks = allowed; other roles = denied

### F7 Application

F7 follows the same pattern:
- `source = "agent"` → audit context, NOT a capability grant
- `capability = ToolSpec.capability` → policy evaluates the TOOL's capability (e.g. READ, WRITE)
- `role = principal.role` → same role as interactive SSH user (principal is inherited, not set by agent)
- Policy: principal.role + READ on filesystem = allowed; principal.role + WRITE on vault = denied (no rule)

### Why Agent ≠ Capability

A capability answers: "WHAT authorization is needed?"

- READ → need permission to read
- WRITE → need permission to write
- EXECUTE → need permission to execute
- NETWORK → need permission for network access
- SECRET → need permission for secret access
- BACKGROUND → need permission for background execution

"AGENT" does not answer "what authorization is needed." It answers "who initiated this." That's a source/context, not a capability.

If we made AGENT a capability, we would need policy rules like:
```
Rule("admin", "AGENT", "filesystem.read_file", allow=True)
```

But this duplicates the existing rule:
```
Rule("admin", "READ", "filesystem.read_file", allow=True)
```

The tool already has a capability. Adding AGENT as an additional capability layer creates confusion about which one governs authorization.

### Decision: No AGENT Capability

AGENT is a **source** (like "background"), not a **capability**. The tool's own capability from CATALOG is what policy evaluates.

---

## 13. Explicit Decision: AGENT Capability

### Analysis

| Option | Pros | Cons |
|--------|------|------|
| AGENT as capability | Explicit marker for agent calls | Duplicates existing tool capabilities; creates confusion about which capability governs |
| AGENT as source | Clean separation (source=who, capability=what); follows F6 pattern | None — this is the correct pattern |
| Both AGENT source + capability | Maximum traceability | Redundant; capability layer adds no security value |

### Decision

**AGENT is a source, NOT a capability.**

- `source = "agent"` in audit entries (like `source = "background"`)
- `capability = ToolSpec.capability` for policy evaluation (tool's own capability)
- `role = principal.role` for policy identity (same as interactive user)

### Proof of No Privilege Escalation

1. Agent tool calls go through `policy.evaluate(role, command, capability, resource)`
2. `role = principal.role` — same as interactive SSH user (inherited from invoking principal)
3. `capability` = tool's capability from CATALOG (e.g. "READ" for filesystem.read_file)
4. Policy rules are the same rules that govern interactive commands
5. Agent gets EXACTLY the same permissions as an interactive admin user
6. No rule grants extra permission based on source="agent"
7. If agent tries vault.write → denied (no admin+WRITE+vault rule for agent source, and even if there were, it would require approval)

### Capability Enum Unchanged

```python
class Capability(str, Enum):
    READ = "READ"
    WRITE = "WRITE"
    EXECUTE = "EXECUTE"
    SECRET = "SECRET"
    NETWORK = "NETWORK"
    BACKGROUND = "BACKGROUND"
    # NO AGENT — not needed
```

---

## 14. Tiny-Model Evaluation Plan

### Candidate Models
- **Cactus Needle** (if available) — ultra-small, tool-calling focused
- **Qwen2.5-0.5B** — smallest Qwen with tool-call support
- **Phi-3.5-mini** — Microsoft small model
- **SmolLM2-1.7B** — HuggingFace small model
- **Gemma-2-2B** — Google small model

### Evaluation Criteria

| Criterion | Weight | Measurement |
|-----------|--------|-------------|
| Correct tool selection | 20% | % of test cases where model selects the right tool |
| Argument extraction | 15% | % of arguments correctly extracted |
| Missing argument detection | 10% | % of cases where model identifies missing required args |
| Invalid tool rejection | 10% | % of cases where model doesn't call non-existent tools |
| Malformed output handling | 10% | % of outputs that are parseable JSON |
| Policy denial handling | 5% | Model gracefully handles tool denial |
| Tool error recovery | 5% | Model adjusts after tool error |
| Multi-step reasoning | 10% | Correct multi-tool sequences |
| Latency | 5% | < 500ms per inference on CPU |
| Memory footprint | 5% | < 2GB RAM for inference |

### Benchmark Setup

1. **FakeModelProvider**: Deterministic responses for CI (100% pass rate baseline)
2. **Synthetic test suite**: 50-100 predefined scenarios with expected tool calls
3. **Real model evaluation**: Run against actual tiny model on vox-space
4. **Comparison**: FakeModelProvider = 100%, real model measured against that

---

## 15. Daemon Architecture Proposal

### When to Daemonize

Only after:
1. Model evaluation proves suitable (benchmark > 80% tool selection accuracy)
2. Provider abstraction is stable (F7-P1 through F7-P4 complete)
3. Latency/throughput requirements justify persistent process

### Daemon Design

```
uta-local-intel.service (systemd simple, not oneshot)
├── Model lifecycle (load, unload, reload)
├── Request queue (inference requests via Unix socket)
├── Inference engine (llama-cpp-python or ONNX runtime)
├── Health check endpoint (localhost only)
└── IPC via Unix socket or HTTP localhost
```

### Daemon Constraints

| Resource | Constraint |
|----------|-----------|
| User | Dedicated `uta-intel` user, NOT root |
| Filesystem | Read-only except model dir and /tmp |
| Network | Localhost only (Unix socket preferred) |
| Vault | No access |
| Credentials | No access |
| Shell | No access |
| Docker | No access |
| systemd | MemoryMax=2G, CPUQuota=80% |

### Daemon is NOT a Security Authority

The daemon provides inference. It does NOT:
- Make authorization decisions
- Access tools directly
- Bypass Gate policy
- Hold credentials
- Control execution

---

## 16. Test Strategy

### 16.1 Unit Tests (test_f7_agent.py)

- ModelProvider protocol compliance
- FakeModelProvider: deterministic responses
- ModelRequest/ModelResponse serialization
- Tool-call parsing (valid JSON, XML, malformed)
- Tool-call schema validation (valid args, missing required, wrong types)
- Session: message append, trajectory recording, secret safety
- Trajectory: append-only, no secrets in export
- AgentLoop: single-turn (text response), multi-turn (tool calls)
- AgentLoop: max iterations enforced
- AgentLoop: timeout enforced
- AgentLoop: cancellation works
- AgentLoop: provider failure recovery
- AgentLoop: malformed tool call handling
- AgentLoop: unknown tool handling
- AgentLoop: policy denial handling

### 16.2 Integration Tests

- Agent tool call → Gate → Policy → MCP → result
- Agent tool call denied by policy
- Agent write → approval required → approval flow
- Agent tool call with invalid arguments → schema rejection
- Agent tool call to unknown tool → catalog rejection
- Agent loop: provider failure → error recovery
- Agent loop: cancellation mid-tool-call
- Agent loop: audit entries created correctly
- Agent loop: trajectory matches audit

### 16.3 Security Regression Tests

- Model cannot bypass policy (test all deny scenarios)
- Model cannot access vault without rules
- Model cannot execute shell commands
- Model cannot write without approval
- Credentials never appear in session/trajectory
- Tool results sanitized before feeding to model
- Agent loop max iterations enforced
- Agent loop timeout enforced
- No privilege escalation beyond admin interactive
- Audit chain integrity after agent operations
- Source="agent" in audit entries (not granting extra privilege)
- Agent role=principal.role, identical to interactive user role

### 16.4 Existing Test Regression

- All 304 existing tests must pass unchanged
- F7 adds tests, does not modify existing test behavior

---

## 17. Files/Modules Expected to Change

### New Files (F7)

| File | Purpose | Est. Lines |
|------|---------|------------|
| `gate/agent/__init__.py` | Package init | ~5 |
| `gate/agent/provider.py` | ModelProvider protocol, ModelRequest, ModelResponse, ToolCall, ToolDefinition, ModelMetadata, TokenUsage | ~120 |
| `gate/agent/providers/__init__.py` | Package init | ~5 |
| `gate/agent/providers/fake.py` | FakeModelProvider for testing | ~80 |
| `gate/agent/providers/local.py` | LocalModelProvider (llama-cpp wrapper) | ~150 |
| `gate/agent/session.py` | Session, Trajectory, TrajectoryEntry | ~100 |
| `gate/agent/loop.py` | AgentLoop — multi-turn inference | ~200 |
| `gate/agent/tool_call.py` | Tool-call parsing, validation, schema generation from CATALOG | ~100 |
| `gate/agent/config.py` | AgentConfig | ~40 |
| `gate/tests/test_f7_agent.py` | Tests for F7 | ~400 |

### Modified Files (F7)

| File | Change |
|------|--------|
| `gate/ingress/schemas.py` | Add "agent" to `_SOURCES` |
| `gate/core.py` | Add `handle_agent_tool_call()` method |
| `gate/config.py` | Add agent config fields (model_path, max_iterations, etc.) |
| `gate/factory.py` | Wire agent provider into Gate construction |
| `gate/mcp/catalog.py` | Add `argument_schema` to ToolSpec (JSON Schema per tool) |

### NOT Modified

| File | Reason |
|------|--------|
| `gate/policy/classes.py` | No new capability needed |
| `gate/policy/engine.py` | No new policy rules needed for agent |
| `gate/cloud/*` | Cloud execution remains independent |
| `gate/mcp/client.py` | MCP execution boundary unchanged |
| `gate/mcp/servers/*` | MCP server implementations unchanged |
| `gate/audit/*` | Audit mechanism unchanged |
| `gate/approval/*` | Approval mechanism unchanged |
| `gate/heartbeat/*` | Heartbeat unchanged |
| `gate/auth/*` | Auth mechanism unchanged |

---

---

## 19. Principal Inheritance (MUST)

### Rule

**Agent MUST inherit Principal from the invoking session. Agent MUST NOT default to admin or elevate privileges.**

The Principal authenticated for the original user request (via AuthProvider.authenticate()) is the same Principal that flows into the agent loop. If no principal exists then deny. No exceptions.

**Agent identity/source never grants privilege.** Agent only inherits authority from the invoking Principal. The string "agent" is not a role, not a capability, and not a credential.

### Why This Matters

Without this rule, it would be tempting to create Principal(id="agent", role="admin") as a convenience shortcut. This would silently elevate the agent to admin regardless of who actually invoked it. A non-admin user would get admin execution behind the scenes.

The policy engine evaluates `principal.role`. If the invoking principal has role=viewer, the agent has role=viewer. The agent does not get admin because it is an agent. It only gets what the invoking principal has.

### Implementation

```python
class AgentLoop:
    def __init__(self, *, principal: Principal, ...):
        # principal is REQUIRED, not optional
        # This is the exact Principal from AuthProvider.authenticate()
        # Not Principal(id="agent", role="admin")
        # Not Principal(id="agent", role=principal.role)
        # Just: the user's own principal, unchanged
        self._principal = principal

    def _execute_tool_call(self, ...):
        result = self._gate.handle_agent_tool_call(
            principal=self._principal,  # user principal, passed through
            server_id=...,
            tool_id=...,
            args=...,
            request_id=...,
        )
```

### Gate.handle_agent_tool_call() Principal is Mandatory

```python
def handle_agent_tool_call(
    self,
    *,
    principal: Principal,      # REQUIRED, never optional, never defaulted
    server_id: str,
    tool_id: str,
    args: dict,
    request_id: str,
) -> McpResult:
    if principal is None:
        raise AuthError("agent tool call requires principal")
    # principal is the user's Principal from AuthProvider.authenticate()
    # We do NOT create a new Principal here
    # We do NOT modify principal.role
```

### Policy Evaluation

```python
decision = self.policy.evaluate(
    role=principal.role,      # principal.role, NOT role="admin"
    command=spec.command_id,
    capability=spec.capability,
    resource=resource,
)
```

`principal.role` is whatever the invoking user's role is. If invoking user is viewer, agent gets viewer permissions. If invoking user is admin, agent gets admin permissions. The agent does not set or override the role.

### What Does NOT Happen

```python
# WRONG - agent does not create its own principal
principal = Principal(id="agent", role="admin")  # WRONG

# WRONG - agent does not escalate role
principal = Principal(id="agent", role="admin")  # WRONG
gate.handle_agent_tool_call(principal=principal, ...)

# WRONG - agent does not override role
principal = Principal(id=original.id, role="admin")  # role changed
```

### Test Cases

1. Agent invoked by admin principal -> agent gets admin permissions (because invoking principal IS admin)
2. Agent invoked by viewer principal -> agent gets viewer permissions (restricted, because invoking principal is viewer)
3. Agent invoked with no principal -> denied, no execution
4. Agent invoked by expired/invalid principal -> denied
5. Agent invoked by admin, model requests vault.write -> denied if no WRITE+vault rule for admin
6. Agent invoked by viewer, model requests filesystem.write -> denied (viewer lacks WRITE)

---

## 20. Approval / Resume Lifecycle

F7 does not create a new approval mechanism. It reuses the existing F2 ApprovalStore and state machine entirely. The only F7 addition is: (1) agent state persistence for resumability, (2) the loop logic that suspends and resumes.

### 20.1 State Diagram

```
                    AgentLoop States

  user message --> RUNNING
                    |
                    +--> model text response --> COMPLETED
                    |
                    +--> model tool_call
                         |
                         +--> policy: allowed --> execute --> RUNNING
                         |
                         +--> policy: denied --> error to model --> RUNNING
                         |
                         +--> policy: needs_approval
                              |
                              +--> WAITING_APPROVAL
                                   |
                                   +--> approved --> execute --> RUNNING
                                   |
                                   +--> denied --> error --> RUNNING
                                   |
                                   +--> expired --> cleanup --> RUNNING

  CANCELLED (user abort)

F2 ApprovalStore states (unchanged):
  NOT_REQUIRED --> PENDING --> APPROVED --> CONSUMED
                     |            |
                     +--> DENIED  |
                     +--> EXPIRED -+
```

### 20.2 Identity Relationship

F7 does not introduce unnecessary duplicate IDs. The existing F2 identifiers are reused:

```
session_id                          # Agent session identity (new in F7, for agent state only)
  +-- request_id                    # F2 Gate request identity (existing)
       +-- command                  # ToolSpec.command_id (existing)
       +-- capability               # ToolSpec.capability (existing)
       +-- arg_hash                 # SHA-256 of sorted args (existing, in Approval)
       +-- approval_id              # Approval.id (existing, auto-generated UUID)
```

| Identifier | Created By | Used For |
|-----------|-----------|----------|
| session_id | AgentLoop | Agent state persistence, trajectory grouping. New in F7. NOT used for approval. |
| request_id | AgentLoop / Gate | Stable tool-call identity. Used as ApprovalStore key. Links approval to exact tool call. |
| approval_id | ApprovalStore.create() | Approval record identity. Used for human-facing approval UI. Retrieved via request_id. |
| tool_call_id | Model response | Model's tool-call correlation. Mapped to request_id 1:1 by AgentLoop. NOT used for approval. |

**request_id is the stable identity.** It persists from the moment the model produces a tool call, through approval creation, human decision, consume check, and MCP execution. No re-creation, no re-issuance.

approval_id is a secondary identifier derived from request_id via ApprovalStore.get(request_id).id. It is NOT a separate identity -- it is a lookup convenience for the approval UI.

### 20.3 Existing F2 Infrastructure (Reused As-Is)

| Component | Location | Used For |
|-----------|----------|----------|
| ApprovalStore.create() | approval/store.py | Creates PENDING approval bound to (request_id, command, capability, arg_hash, requester) |
| ApprovalStore.approve() | approval/store.py | Transitions PENDING -> APPROVED |
| ApprovalStore.deny() | approval/store.py | Transitions PENDING -> DENIED |
| ApprovalStore.consume() | approval/store.py | Transitions APPROVED -> CONSUMED. Verifies binding: command, capability, arg_hash, requester. Fails if binding mismatch. |
| ApprovalStore.get() | approval/store.py | Retrieves approval by request_id. Auto-reaps expired PENDING. |
| Approval dataclass | approval/store.py | Bound to (request_id, command, capability, arg_hash, requester, expires_at) |
| ApprovalState | approval/state_machine.py | NOT_REQUIRED, PENDING, APPROVED, DENIED, EXPIRED, CONSUMED |
| transition() | approval/state_machine.py | Validates state transitions |

**No new classes, no new methods, no new state machine.**

### 20.4 Pseudocode: handle_agent_tool_call()

```python
def handle_agent_tool_call(
    self,
    *,
    principal: Principal,
    server_id: str,
    tool_id: str,
    args: dict,
    request_id: str,
) -> McpResult:
    # 1. Catalog lookup
    spec = tool_lookup(server_id, tool_id)
    if spec is None:
        raise ValidationError(f"unknown tool: {server_id}.{tool_id}")

    # 2. Scope validation
    resource = self._mcp_validate_scope_for_agent(server_id, tool_id, args, spec)

    # 3. Policy evaluation
    decision = self.policy.evaluate(
        role=principal.role,
        command=spec.command_id,
        capability=spec.capability,
        resource=resource,
    )

    # 4. Denied
    if not decision.allowed:
        self.audit_log.append({
            "source": "agent",
            "request_id": request_id,
            "command": spec.command_id,
            "capability": spec.capability,
            "principal_id": principal.id,
            "result": "denied",
        })
        return McpResult(status="error", error_class="denied",
                         data={"error": f"policy denied: {decision.reason}"})

    # 5. Approval required -- use existing F2 ApprovalStore
    if decision.needs_approval:
        approval = self.approval_store.create(
            request_id=request_id,
            command=spec.command_id,
            capability=spec.capability,
            args=args,
            requester=principal.id,
        )
        self.audit_log.append({
            "source": "agent",
            "request_id": request_id,
            "command": spec.command_id,
            "capability": spec.capability,
            "principal_id": principal.id,
            "result": "approval_requested",
            "approval_id": approval.id,
        })
        return McpResult(
            status="approval_required",
            data={
                "approval_id": approval.id,
                "request_id": request_id,
                "command": spec.command_id,
            },
        )

    # 6. Execute via MCP
    try:
        result = self.mcp_client.call(server_id, tool_id, args)
    except Exception as e:
        self.audit_log.append({
            "source": "agent",
            "request_id": request_id,
            "command": spec.command_id,
            "capability": spec.capability,
            "principal_id": principal.id,
            "result": "execution_error",
        })
        return McpResult(status="error", error_class="mcp_error",
                         data={"error": str(e)})

    # 7. Audit success
    self.audit_log.append({
        "source": "agent",
        "request_id": request_id,
        "command": spec.command_id,
        "capability": spec.capability,
        "principal_id": principal.id,
        "result": "executed",
    })
    return result
```

### 20.5 Pseudocode: AgentLoop -- Suspension and Resumption

```python
class AgentLoop:
    def __init__(
        self,
        *,
        principal: Principal,
        session_id: str,
        gate: Gate,
        provider: ModelProvider,
        approval_store: ApprovalStore,
        audit_log: AuditLog,
        max_iterations: int = 50,
    ):
        self._principal = principal
        self._session_id = session_id
        self._gate = gate
        self._provider = provider
        self._approval_store = approval_store
        self._audit_log = audit_log
        self._max_iterations = max_iterations
        self._trajectory = []
        self._state = AgentState.RUNNING

    def run(self, user_message: str) -> AgentResult:
        if user_message:
            self._trajectory.append(TrajectoryEntry(role="user", content=user_message))

        for _ in range(self._max_iterations):
            response = self._provider.complete(self._build_request())

            if response.text and not response.tool_calls:
                self._trajectory.append(TrajectoryEntry(role="assistant", content=response.text))
                self._state = AgentState.COMPLETED
                return AgentResult(state=self._state, trajectory=self._trajectory)

            if response.tool_calls:
                for tool_call in response.tool_calls:
                    result = self._execute_tool_call(tool_call)

                    if result.status == "approval_required":
                        self._state = AgentState.WAITING_APPROVAL
                        self._persist_state(pending_request_id=tool_call.request_id)
                        return AgentResult(
                            state=self._state,
                            trajectory=self._trajectory,
                            pending_request_id=tool_call.request_id,
                        )

                    self._trajectory.append(
                        TrajectoryEntry(role="tool", tool_call_id=tool_call.id, content=result.data)
                    )

        self._state = AgentState.FAILED
        return AgentResult(state=self._state, trajectory=self._trajectory)

    def _execute_tool_call(self, tool_call: ToolCall) -> McpResult:
        self._validate_tool_call(tool_call)
        return self._gate.handle_agent_tool_call(
            principal=self._principal,
            server_id=tool_call.server_id,
            tool_id=tool_call.tool_id,
            args=tool_call.args,
            request_id=tool_call.request_id,
        )

    def resume(self) -> AgentResult:
        state = self._load_state()
        request_id = state["pending_request_id"]
        approval = self._approval_store.get(request_id)

        if approval is None:
            raise RuntimeError(f"no approval found for request_id={request_id}")

        if approval.state == ApprovalState.APPROVED:
            self._approval_store.consume(
                request_id=request_id,
                command=approval.command,
                capability=approval.capability,
                args=approval.args,
                requester=approval.requester,
            )
            result = self._gate.execute_approved_tool_call(
                principal=self._principal,
                request_id=request_id,
            )
            self._trajectory.append(
                TrajectoryEntry(role="tool", tool_call_id=request_id, content=result.data)
            )
            self._state = AgentState.RUNNING
            self._clear_pending_state()
            return self.run("")

        if approval.state == ApprovalState.DENIED:
            self._trajectory.append(
                TrajectoryEntry(role="tool", tool_call_id=request_id,
                                content={"denied": True, "reason": "approval denied by user"})
            )
            self._state = AgentState.RUNNING
            self._clear_pending_state()
            return self.run("")

        if approval.state == ApprovalState.EXPIRED:
            self._audit_log.append({
                "source": "agent", "request_id": request_id,
                "result": "approval_expired", "principal_id": self._principal.id,
            })
            self._trajectory.append(
                TrajectoryEntry(role="tool", tool_call_id=request_id,
                                content={"expired": True, "reason": "approval TTL expired"})
            )
            self._state = AgentState.RUNNING
            self._clear_pending_state()
            return self.run("")

        raise RuntimeError(f"unexpected approval state: {approval.state}")
```

### 20.6 Resume Security -- Binding Invariants

On resume, ApprovalStore.consume() verifies ALL of the following. If ANY fails, execution is rejected.

```python
# Inside ApprovalStore.consume():
binds = (
    a.command == command
    and a.capability == capability
    and a.arg_hash == arg_hash(args)
    and a.requester == requester
)
```

| Binding Field | What It Prevents |
|---------------|-----------------|
| command | Approved filesystem.read executed as filesystem.write |
| capability | Approved READ executed with WRITE capability |
| arg_hash | Approved path=/tmp/a executed with path=/etc/shadow |
| requester | User A approval used for User B execution |

**If arguments change after approval:** New args produce different arg_hash. consume() raises InvalidTransition. Agent must re-request approval with new request_id.

**If the approval is denied or expired:** consume() raises InvalidTransition (state is not APPROVED). Execution is rejected.

**If the tool/server changes:** Different command value. consume() raises InvalidTransition.

**The approval is NOT transferable.** It is bound to exactly one tool call with exactly one set of arguments for exactly one principal.

### 20.7 Resume Security -- Additional Invariants

Beyond the ApprovalStore binding check, the resume path also verifies:

1. **Same Principal** -- AgentState.principal_id must match the current principal. If principal changed (e.g., session expired), resume is rejected.
2. **Same session** -- AgentState.session_id must match. Agent state cannot be loaded by a different session.
3. **Scope re-validation** -- Gate.execute_approved_tool_call() re-validates scope via _mcp_validate_scope_for_agent(). If the tool's server or scope has changed, execution is rejected.
4. **Policy re-evaluation** -- Gate.execute_approved_tool_call() re-evaluates policy. If rules changed while agent was suspended, execution follows the new policy.

### 20.8 AgentState -- Minimal Paused State

```python
class AgentState(str, Enum):
    RUNNING = "RUNNING"
    WAITING_APPROVAL = "WAITING_APPROVAL"
    COMPLETED = "COMPLETED"
    FAILED = "FAILED"
    CANCELLED = "CANCELLED"

@dataclass
class PersistedAgentState:
    session_id: str
    principal_id: str
    principal_role: str
    trajectory: list[dict]
    pending_request_id: str | None
    paused_at: float | None
    created_at: float
```

Stored at `/data/vault/agents/{session_id}/state.json`. Written when agent suspends (WAITING_APPROVAL). Read when agent resumes. Deleted after agent completes or fails.

**The pending_request_id is the only link to the approval.** The approval record itself lives in ApprovalStore (F2), not in AgentState. AgentState does not duplicate approval data.

### 20.9 Approval Resume Flow -- Complete Sequence

```
AgentLoop.run(user_message)
    |
    v
ModelProvider.complete(request)
    |
    v
ModelResponse(tool_calls=[ToolCall(...)])
    |
    v
AgentLoop._execute_tool_call(tool_call)
    |
    +-- F7 validates ToolCall structure
    |
    v
Gate.handle_agent_tool_call(principal, server_id, tool_id, args, request_id)
    |
    +-- Catalog lookup -> ToolSpec
    +-- Scope validation
    +-- PolicyEngine.evaluate(role=principal.role, command, capability, resource)
    |
    v
Decision.needs_approval == True
    |
    +-- ApprovalStore.create(              <-- F2, existing
    |      request_id=request_id,
    |      command=spec.command_id,
    |      capability=spec.capability,
    |      args=args,
    |      requester=principal.id,
    |   ) -> Approval(id=approval_id, state=PENDING)
    |
    +-- AuditLog.append({                  <-- F2, existing
    |      source="agent",
    |      result="approval_requested",
    |      ...
    |   })
    |
    +-- AgentLoop._persist_state(          <-- F7, new
    |      pending_request_id=request_id,
    |   ) -> /data/vault/agents/{session_id}/state.json
    |
    +-- AgentLoop._state = WAITING_APPROVAL
    |
    v
AgentLoop returns AgentResult(state=WAITING_APPROVAL)
    |
    |  ... time passes, user reviews ...
    |
    v
User approves via existing approval mechanism:
    ApprovalStore.approve(request_id, approved_by=user_id)
    |
    v
AgentLoop.resume()                        <-- F7, new
    |
    +-- Load AgentState from disk
    +-- ApprovalStore.get(pending_request_id)  <-- F2, existing
    |
    v
approval.state == APPROVED
    |
    +-- ApprovalStore.consume(             <-- F2, existing
    |      request_id=request_id,
    |      command=approval.command,
    |      capability=approval.capability,
    |      args=approval.args,
    |      requester=approval.requester,
    |   )
    |   -> Binding check: command, capability, arg_hash, requester
    |   -> If mismatch -> InvalidTransition, execution rejected
    |   -> If ok -> state = CONSUMED
    |
    +-- Gate.execute_approved_tool_call(   <-- F7, new (thin wrapper)
    |      principal=principal,
    |      request_id=request_id,
    |   )
    |   -> Scope re-validation
    |   -> Policy re-evaluation
    |   -> McpClient.call(server_id, tool_id, args)
    |
    +-- AuditLog.append({                  <-- F2, existing
    |      source="agent",
    |      result="executed",
    |      ...
    |   })
    |
    +-- TrajectoryEntry(role="tool", content=result)
    |
    +-- AgentLoop._clear_pending_state()   <-- F7, new
    |   -> Remove /data/vault/agents/{session_id}/state.json
    |
    +-- AgentLoop._state = RUNNING
    |
    v
AgentLoop.run("") -> continues with existing trajectory
    |
    v
ModelProvider.complete(trajectory) -> next iteration
```

### 20.10 Audit Events -- Approval Lifecycle

All audit entries use the existing F2 AuditLog (hash-chain). No parallel log.

| Event | source | result | approval_id | When |
|-------|--------|--------|-------------|------|
| tool requested | agent | allowed | -- | Policy allowed, no approval needed |
| tool requested | agent | denied | -- | Policy denied |
| approval requested | agent | approval_requested | yes | Policy needs approval, ApprovalStore.create() |
| approval approved | -- | approval_approved | yes | User approved (via existing approval mechanism) |
| approval denied | -- | approval_denied | yes | User denied |
| approval expired | agent | approval_expired | yes | TTL expired while waiting |
| tool executed | agent | executed | -- | Successful MCP execution |
| tool execution failed | agent | execution_error | -- | MCP execution error |

All entries include: source="agent", command=spec.command_id, capability=spec.capability, principal_id=principal.id, role=principal.role.

**Never:** capability="AGENT" or source=anything_else.

### 20.11 Failure / Expiry / Cancellation

| Scenario | Behavior |
|----------|----------|
| Approval denied | AgentState cleared. Model receives denial in trajectory. Agent loop continues. |
| Approval expired | ApprovalStore.get() auto-reaps to EXPIRED. Agent state cleaned. Audit logged. Model receives expiry in trajectory. Agent loop continues. |
| Arguments changed after approval | ApprovalStore.consume() raises InvalidTransition (arg_hash mismatch). Agent must create new request_id and re-request approval. |
| Principal changed/expired | Resume rejected. Agent state cleaned. Audit logged. |
| Agent cancelled by user | AgentState set to CANCELLED. Pending approval remains PENDING (cleaned by existing TTL). Audit logged. |
| Max iterations reached | AgentState set to FAILED. Any pending approval cleaned. Audit logged. |
| Provider error | AgentState set to FAILED. Error in trajectory. |


## 21. Source=Agent as Provenance-Only

### Rule

**source=agent is a provenance marker ONLY. It is never used for policy evaluation, authorization, or privilege escalation.**

### What Provenance-Only Means

Source answers ONE question: Who or what initiated this action?

| Source    | Meaning                        | Used for Policy? |
|-----------|--------------------------------|-----------------|
| interactive | User typed the command directly | NO |
| background  | Heartbeat daemon initiated      | NO |
| agent       | Agent loop (model) initiated    | NO |

Source is recorded in: audit log entries, trajectory entries, EventBus events.

Source is NEVER consumed by: PolicyEngine.evaluate(), ApprovalStore, any authorization check.

### Why Source Matters

Even though source is not used for authorization, it is critical for:

1. Audit traceability: When reviewing logs, you need to know if a tool call came from a human, heartbeat, or agent
2. Forensics: If an agent misbehaves, you can trace its exact trajectory
3. Debugging: Distinguishing agent-initiated vs human-initiated tool calls in error analysis
4. Accountability: The requester field in approval records is the human's ID (from Principal), not agent

### Implementation

```python
audit_log.append(
    command="filesystem.write_file",
    capability="WRITE",
    resource="/data/report.md",
    role=principal.role,            # used for authorization
    source="agent",                 # provenance only, NOT used for auth
    result="allowed",
    principal_id=principal.id,      # human ID, NOT agent
)
```

### What Does NOT Happen

```python
# WRONG - policy never checks source
engine.evaluate(..., source="agent")

# WRONG - no policy rule references source
Rule(role="admin", source="agent", command="filesystem.write_file", allow=True)  # WRONG

# WRONG - agent identity is not stored as principal
principal = Principal(id="agent", role="admin")  # WRONG
```

### Relationship to Section 19 (Principal Inheritance)

Section 19 says: agent inherits the user's Principal.
Section 21 says: source=agent is provenance only.

These are complementary:

- Principal: WHO is authorized (the human user's identity and role)
- Source: WHAT initiated the action (agent loop, for audit traceability)

Both are recorded in audit entries. Only Principal is used for authorization.

---

## 18. Decision Gate

### A. What Changed from Previous F7-P0

| Area | Previous (Original) | Corrected | Why |
|------|---------------------|-----------|-----|
| AGENT capability | Proposed adding `AGENT = "AGENT"` to Capability enum | **Not needed.** Agent is a source, not a capability | F6 shows source/capability distinction; agent uses tool's own capability |
| Policy rules | Implicit agent-specific rules | No new rules. Agent uses existing MCP tool rules | Agent role=principal.role, same permissions as interactive user |
| Tool schema exposure | Included capability in ToolDefinition | Capability NOT exposed to model | Model doesn't need to know authorization details |
| Gate integration | Agent calls Gate through same handle_mcp | New `handle_agent_tool_call()` — internal method, no HTTP validation | Agent loop is internal, but still goes through policy/approval/audit |
| Source convention | Not clearly defined | `source = "agent"` (like `source = "background"`) | Follows F6 precedent for audit traceability |
| Security analysis | General | Detailed threat model with 11 specific threats | User requested thorough security boundary analysis |
| Daemon constraints | Basic | Detailed resource constraints (user, filesystem, network, vault, shell) | Security hardened before implementation |

### B. Why Each Change Was Necessary

1. **AGENT capability removed**: Creating a new capability by analogy with BACKGROUND would create confusion. BACKGROUND is used because heartbeat tasks are specific named operations. Agent calls use existing tools with existing capabilities. No new capability is needed.

2. **Capability not exposed to model**: If the model knows that `filesystem.read_file` has capability="READ", it could reason about capabilities and try to find bypasses. By hiding capability, the model only knows tool name + description + parameters.

3. **Agent source convention**: source="agent" provides audit traceability without granting privilege. This follows the exact pattern F6 established with source="background". Source is never used for authorization.

4. **handle_agent_tool_call()**: Separates the internal agent path from the external HTTP path. Both go through the same policy/approval/audit, but the internal path skips HTTP schema validation (agent loop already validated).

### C. AGENT Capability Decision

**Not needed.** Agent is a source/context (like "background"), not a capability. Tool calls use the tool's own capability from CATALOG. Policy evaluates (role, command, tool_capability, resource) — agent identity is irrelevant to authorization.

### D. How F7 Identifies Agent-Originated Requests Without Extra Privilege

- `source = "agent"` in audit entries (traceability only)
- `role = principal.role` for policy evaluation (same as interactive user)
- `capability = ToolSpec.capability` (tool's own capability)
- No policy rule references source="agent" (no source-specific rules)
- Agent gets identical permissions to interactive admin user

### E. How Tool Calls Enter F2/F3

1. Agent loop parses model output → `ToolCall(server_id, tool_id, arguments)`
2. Agent loop calls `Gate.handle_agent_tool_call(principal, server_id, tool_id, args, request_id)`
3. Gate does catalog lookup → `ToolSpec`
4. Gate does scope validation
5. Gate does `policy.evaluate(role, command, ToolSpec.capability, resource)`
6. If allowed → `McpClient.call()` → `McpResult`
7. Result sanitized → fed back to model
8. Audit entry created

### F. How Trajectory Remains Secret-Sensitive

- Session stores messages, tool calls, tool results
- Tool results sanitized before storage:
  - Vault content redacted (only secret_id recorded)
  - No credential patterns in file content
  - Error messages scrubbed
- Trajectory export for audit: sanitized view
- No API keys, no vault credentials, no raw secrets ever enter session
- Same redaction rules as existing AuditLog

### G. How Tiny Local Models Plug In Without Changing Gate Security

1. `LocalModelProvider` implements `ModelProvider` protocol
2. Plugs into `AgentLoop` via dependency injection
3. Gate security pipeline unchanged:
   - Policy engine same
   - MCP boundary same
   - Audit same
   - Approval same
4. Model only affects: what tool calls are generated
5. Gate decides: whether those calls are allowed
6. Model change = different ToolCall generation, same authorization

### H. Recommended Next Step

**F7-P1: ModelProvider abstraction + FakeModelProvider.**

Rationale:
- P0 architecture is internally consistent with F6
- No AGENT capability needed (simplifies implementation)
- Agent source convention established
- Gate integration method designed
- Security boundaries verified
- FakeModelProvider allows testing entire agent loop without real model
- No model download/install needed yet

F7-P1 deliverables:
1. `gate/agent/provider.py` — ModelProvider protocol, ModelRequest, ModelResponse, ToolCall, ToolDefinition
2. `gate/agent/providers/fake.py` — FakeModelProvider with deterministic responses
3. `gate/agent/tool_call.py` — tool-call parsing + schema validation
4. `gate/tests/test_f7_agent.py` — unit tests for provider + tool-call parsing
5. Add "agent" to `_SOURCES` in ingress/schemas.py

---

## Invariant Verification

| Invariant | Status |
|-----------|--------|
| deny-by-default preserved | ✅ Agent uses existing MCP rules, no new allow rules |
| policy enforcement preserved | ✅ policy.evaluate() called for every tool call |
| approval enforcement preserved | ✅ Write operations still require approval |
| audit integrity preserved | ✅ source="agent" entries in same audit log |
| secret isolation preserved | ✅ No secrets in session/trajectory |
| MCP boundary preserved | ✅ Tool calls go through McpClient |
| runtime isolation preserved | ✅ No direct filesystem/shell access for model |
| no shell preserved | ✅ Agent cannot execute shell commands |
| no arbitrary execution preserved | ✅ Only CATALOG tools callable |
| | no privilege escalation through agent/model identity | Agent role=principal.role = interactive user, no source-based privilege |

---

## F7-P0 CORRECTED = COMPLETE

Awaiting review before proceeding to F7-P1.
