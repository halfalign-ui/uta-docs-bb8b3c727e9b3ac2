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

**F7 integration**: Model tool calls use role="admin" (same as interactive SSH user). Policy decides based on (role, tool's capability, command). Model identity does not influence the decision.

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

### 5.3 Gate Integration — No Parallel Authorization

The agent loop does NOT bypass Gate. A new internal method on Gate:

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
    """Internal tool-call entry for agent loop.
    
    Same pipeline as handle_mcp, without HTTP schema validation.
    Agent loop already validated the request structure.
    """
    # 1. Catalog lookup
    spec = tool_lookup(server_id, tool_id)
    if spec is None:
        raise ValidationError(f"unknown tool: {server_id}.{tool_id}")

    # 2. Scope validation
    resource = self._mcp_validate_scope_for_agent(server_id, tool_id, args, spec)

    # 3. Policy evaluation — uses tool's capability, NOT agent capability
    decision = self.policy.evaluate(
        role=principal.role,
        command=spec.command_id,
        capability=spec.capability,  # <-- tool's capability from CATALOG
        resource=resource,
    )

    # 4. Deny → audit + return error
    if not decision.allowed:
        self._audit_mcp_agent(server_id, tool_id, args, decision, "denied", spec)
        return McpResult(status="error", error_class="denied",
                         data={"error": f"policy denied: {decision.reason}"})

    # 5. Approval required → return approval signal
    if decision.needs_approval:
        return McpResult(status="error", error_class="approval_required",
                         data={"approval_required": True,
                               "request_id": request_id,
                               "command": spec.command_id})

    # 6. Execute via MCP
    try:
        result = self.mcp_client.call(server_id, tool_id, args)
    except Exception as e:
        self._audit_mcp_agent(server_id, tool_id, args, decision, "error", spec)
        return McpResult(status="error", error_class="mcp_error",
                         data={"error": str(e)})

    # 7. Audit success
    self._audit_mcp_agent(server_id, tool_id, args, decision, "executed", spec,
                          result_status=result.status)
    return result
```

**Critical**: `capability=spec.capability` — the capability comes from the ToolSpec in CATALOG, NOT from the agent. The agent is never the source of authority.

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
| Model tries to escalate via identity | Agent role = admin same as interactive, no extra privilege |
| Model tries to use agent source for privilege | Source="agent" is audit-only, policy uses tool capability |

---

## 12. Source/Context vs Capability Analysis

### F6 Precedent

F6 established:
- `source = "background"` → audit context, NOT a capability grant
- `capability = BACKGROUND` → policy evaluates this
- `role = "admin"` → who is requesting
- Policy: admin + BACKGROUND on specific tasks = allowed; user = denied

### F7 Application

F7 follows the same pattern:
- `source = "agent"` → audit context, NOT a capability grant
- `capability = ToolSpec.capability` → policy evaluates the TOOL's capability (e.g. READ, WRITE)
- `role = "admin"` → same role as interactive SSH user
- Policy: admin + READ on filesystem = allowed; admin + WRITE on vault = denied (no rule)

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
- `role = "admin"` for policy identity (same as interactive user)

### Proof of No Privilege Escalation

1. Agent tool calls go through `policy.evaluate(role, command, capability, resource)`
2. `role = "admin"` — same as interactive SSH user
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
- Agent role identical to interactive admin role

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

## 18. Decision Gate

### A. What Changed from Previous F7-P0

| Area | Previous (Original) | Corrected | Why |
|------|---------------------|-----------|-----|
| AGENT capability | Proposed adding `AGENT = "AGENT"` to Capability enum | **Not needed.** Agent is a source, not a capability | F6 shows source/capability distinction; agent uses tool's own capability |
| Policy rules | Implicit agent-specific rules | No new rules. Agent uses existing MCP tool rules | Agent role=admin gets same permissions as interactive |
| Tool schema exposure | Included capability in ToolDefinition | Capability NOT exposed to model | Model doesn't need to know authorization details |
| Gate integration | Agent calls Gate through same handle_mcp | New `handle_agent_tool_call()` — internal method, no HTTP validation | Agent loop is internal, but still goes through policy/approval/audit |
| Source convention | Not clearly defined | `source = "agent"` (like `source = "background"`) | Follows F6 precedent for audit traceability |
| Security analysis | General | Detailed threat model with 11 specific threats | User requested thorough security boundary analysis |
| Daemon constraints | Basic | Detailed resource constraints (user, filesystem, network, vault, shell) | Security hardened before implementation |

### B. Why Each Change Was Necessary

1. **AGENT capability removed**: Creating a new capability by analogy with BACKGROUND would create confusion. BACKGROUND is used because heartbeat tasks are specific named operations. Agent calls use existing tools with existing capabilities. No new capability is needed.

2. **Capability not exposed to model**: If the model knows that `filesystem.read_file` has capability="READ", it could reason about capabilities and try to find bypasses. By hiding capability, the model only knows tool name + description + parameters.

3. **Agent source convention**: Source="agent" provides audit traceability without granting privilege. This follows the exact pattern F6 established with source="background".

4. **handle_agent_tool_call()**: Separates the internal agent path from the external HTTP path. Both go through the same policy/approval/audit, but the internal path skips HTTP schema validation (agent loop already validated).

### C. AGENT Capability Decision

**Not needed.** Agent is a source/context (like "background"), not a capability. Tool calls use the tool's own capability from CATALOG. Policy evaluates (role, command, tool_capability, resource) — agent identity is irrelevant to authorization.

### D. How F7 Identifies Agent-Originated Requests Without Extra Privilege

- `source = "agent"` in audit entries (traceability only)
- `role = "admin"` for policy evaluation (same as interactive)
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
| no privilege escalation through agent/model identity | ✅ Agent role=admin = interactive admin, no source-based privilege |

---

## F7-P0 CORRECTED = COMPLETE

Awaiting review before proceeding to F7-P1.
