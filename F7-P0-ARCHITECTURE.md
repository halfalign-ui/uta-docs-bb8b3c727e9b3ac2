# F7-P0 — Local Intelligence Runtime: Architecture & Preflight Report

> Status: PREFLIGHT / DESIGN
> Date: 2026-08-18
> Baseline: F6 (304 tests, commit `04dbb59`)
> Author: opencode session

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
| `policy/classes.py` | Capability enum: READ, WRITE, EXECUTE, SECRET, NETWORK, BACKGROUND | Needs AGENT capability |
| `mcp/catalog.py` | Tool catalog: ToolSpec(server_id, tool_id, capability, scope_kind) | Model queries this for available tools |
| `mcp/client.py` | MCP tool execution with timeout, result limits | Agent loop calls this for tool results |
| `ingress/schemas.py` | Request validation: CommandRequest, McpRequest, ApproveRequest | Agent loop uses McpRequest format |
| `events.py` | EventBus for SSE streaming | Agent loop publishes trajectory events |
| `audit/audit.py` | Hash-chain append-only audit log | Agent actions logged here |
| `approval/store.py` | Approval state machine | Write operations require approval |
| `cloud/runtime.py` | Cloud execution: CloudRuntime.execute() | F7 does NOT replace this; F7 uses model providers differently |
| `cloud/provider.py` | ProviderConfig, ModelSpec, PricingInfo | Reusable abstraction pattern for local providers |
| `cloud/config.py` | CloudConfig, QuotaLimits | Pattern for local model config |
| `factory.py` | build_gate() — single construction point | F7 provider wiring goes here |
| `config.py` | Config dataclass, env loading | F7 config fields added here |

---

## 2. F2–F6 Integration Points

### 2.1 Gate Core Pipeline (F2)
```
handle_mcp(principal, body):
  1. validate_mcp_request(body)     → McpRequest
  2. tool_lookup(server_id, tool_id) → ToolSpec
  3. _mcp_validate_scope(req, spec)  → resource (canonical path/label)
  4. policy.evaluate(role, cmd, cap, resource) → Decision
  5. if needs_approval → _mcp_approval_flow()
  6. mcp_client.call(server_id, tool_id, args) → McpResult
  7. audit.append(...)
  8. return 200, {result}
```

**F7 integration**: The agent loop's tool calls enter at step 1. F7 wraps the model's structured output into an McpRequest-compatible format and feeds it through the existing pipeline.

### 2.2 Policy Engine (F2)
```python
Decision = policy.evaluate(
    role="admin",           # or "user"
    command="filesystem.read_file",
    capability="READ",
    resource="/data/some/path",
)
# Decision(allowed=True/False, needs_approval=True/False, reason="...")
```

**F7 integration**: Model tool-call requests use role="admin" (local model = same privilege as interactive SSH). Policy decides whether the tool call is permitted. Model cannot override this.

### 2.3 MCP Catalog (F3)
```python
CATALOG = {
    ("filesystem", "read_file"): ToolSpec(
        server_id="filesystem", tool_id="read_file",
        capability="READ", scope_kind=ScopeKind.PATH,
    ),
    ...
}
```

**F7 integration**: The agent runtime builds the model's tool schema from CATALOG. Model sees available tools + their argument schemas. Model can only call tools that exist in CATALOG.

### 2.4 MCP Client (F3)
```python
result = mcp_client.call("filesystem", "read_file", {"path": "/data/file.txt"})
# McpResult(status="ok", data={...}, duration_ms=123.4)
```

**F7 integration**: Agent loop calls this to execute tool calls. Results are fed back to the model as tool results in the trajectory.

### 2.5 Audit (F2)
```python
audit.append({
    "source": "agent",       # or "dev", "cloud-l2", "worker", "background"
    "request_id": "req_...",
    "command": "filesystem.read_file",
    "capability": "READ",
    "decision": "allowed",
    "result_status": "ok",
    ...
})
```

**F7 integration**: Agent loop logs every tool call. Source = "agent". Trajectory entries are auditable.

### 2.6 EventBus (F2)
```python
events.publish({"type": "agent_turn", "request_id": "...", "tool_calls": [...]})
```

**F7 integration**: Agent loop publishes events for SSE streaming. UI can observe agent thinking, tool calls, results in real-time.

### 2.7 Approval Gate (F2)
```python
if decision.needs_approval:
    # Create approval, wait for user confirmation
    return approval_flow(...)
```

**F7 integration**: If model requests a write operation, approval gate activates. Agent loop pauses, waits for approval, then continues. Model cannot skip approval.

---

## 3. Existing Reusable Abstractions

| Abstraction | Location | F7 Usage |
|-------------|----------|----------|
| `Capability` enum | `policy/classes.py` | Extend with AGENT |
| `ToolSpec` | `mcp/catalog.py` | Model tool schema source |
| `Decision` | `policy/engine.py` | Tool-call authorization |
| `McpResult` | `mcp/client.py` | Tool execution result |
| `McpRequest` | `ingress/schemas.py` | Tool-call request format |
| `EventBus` | `events.py` | Agent trajectory events |
| `AuditLog` | `audit/audit.py` | Agent action audit |
| `ProviderConfig` | `cloud/provider.py` | Pattern for local model config |
| `ModelSpec` | `cloud/provider.py` | Pattern for model capability metadata |
| `CloudConfig` | `cloud/config.py` | Pattern for local model config |
| `ApprovalStore` | `approval/store.py` | Write-operation approval |

---

## 4. Conflicts / Gaps

### 4.1 Capability Enum Gap
Current capabilities: READ, WRITE, EXECUTE, SECRET, NETWORK, BACKGROUND

**Gap**: No AGENT or MODEL_CAPABILITY for agent loop operations. The agent loop itself is not a "tool" — it's an orchestration layer. But tool calls from the agent need a capability marker.

**Resolution**: Add `AGENT = "AGENT"` to Capability. Agent tool calls use AGENT as the capability. Policy rules for AGENT = same as current admin rules for tool-specific capabilities. The agent doesn't get special powers — it uses the same tool permissions as an interactive user.

### 4.2 Gate Entry Point Gap
Current `handle_mcp()` is designed for external HTTP requests (validates schema, checks source, etc.)

**Gap**: Agent loop calls are internal, not HTTP requests. The agent loop shouldn't need to construct full HTTP request bodies.

**Resolution**: Add `Gate.handle_agent_tool_call()` — internal method that:
- Takes a structured tool call (server_id, tool_id, args)
- Bypasses HTTP schema validation (agent loop already validated)
- Still goes through policy → approval → MCP → audit
- Returns McpResult directly

### 4.3 Tool Schema for Model
Model needs to know what tools are available and their argument schemas.

**Gap**: CATALOG has ToolSpec but no JSON Schema for arguments.

**Resolution**: Add `argument_schema: dict` to ToolSpec (or a separate schema registry). Each MCP server already defines its tools via MCP protocol — we can extract schemas from there, or define them statically per tool.

### 4.4 Trajectory State
Model needs conversation history (messages, tool calls, tool results).

**Gap**: No session/trajectory abstraction exists.

**Resolution**: Create `Session` class (see architecture below).

### 4.5 Agent Loop
No multi-turn inference loop exists.

**Resolution**: Create `AgentLoop` class (see architecture below).

### 4.6 Provider Abstraction
Cloud uses `CloudExecutor` which is OpenAI-compatible. Local model would use different runtime (llama-cpp, ONNX, etc.)

**Gap**: No generic provider abstraction.

**Resolution**: Create `ModelProvider` protocol and `ModelRequest`/`ModelResponse` dataclasses.

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
│   ├── tool_call.py          # Tool-call parsing, validation, schema
│   └── config.py             # AgentConfig (model path, max iterations, etc.)
```

### 5.2 Core Abstractions

#### ModelProvider (Protocol)
```python
class ModelProvider(Protocol):
    def complete(self, request: ModelRequest) -> ModelResponse: ...
    def is_available(self) -> bool: ...
    def metadata(self) -> ModelMetadata: ...
```

#### ModelRequest
```python
@dataclass
class ModelRequest:
    messages: list[dict]           # conversation history
    tools: list[ToolDefinition]    # available tool schemas
    max_tokens: int = 1024
    temperature: float = 0.3
    stop: list[str] | None = None
```

#### ModelResponse
```python
@dataclass
class ModelResponse:
    content: str | None            # text response (if any)
    tool_calls: list[ToolCall]     # tool calls (if any)
    finish_reason: str             # "stop", "tool_call", "max_tokens"
    usage: TokenUsage | None = None
    raw: dict | None = None        # provider-specific raw output
```

#### ToolCall
```python
@dataclass
class ToolCall:
    id: str                        # unique call ID
    server_id: str                 # MCP server
    tool_id: str                   # tool name
    arguments: dict                # parsed arguments
```

#### ToolDefinition
```python
@dataclass
class ToolDefinition:
    server_id: str
    tool_id: str
    name: str                      # display name for model
    description: str               # what the tool does
    parameters: dict               # JSON Schema for arguments
    capability: str                # required capability
```

### 5.3 Session / Trajectory

```python
class Session:
    session_id: str
    messages: list[dict]           # message history
    tool_calls: list[ToolCall]     # all tool calls made
    tool_results: list[McpResult]  # all tool results
    metadata: dict                 # session metadata (no secrets!)

class Trajectory:
    """Append-only log of agent actions. Read-only view of session."""
    entries: list[TrajectoryEntry] # ordered list of actions
```

**Security rule**: No credentials, no API keys, no secrets in session or trajectory.

### 5.4 Agent Loop

```python
class AgentLoop:
    def __init__(
        self,
        *,
        provider: ModelProvider,
        gate: Gate,                # F2 Gate for policy + MCP
        session: Session,
        max_iterations: int = 10,
        timeout_seconds: float = 120.0,
    ): ...

    def run(self, user_message: str) -> AgentResult:
        """Run agent loop: user → model → tool calls → gate → results → model → ..."""
        # 1. Add user message to session
        # 2. Loop (max_iterations):
        #    a. Build ModelRequest from session messages + available tools
        #    b. provider.complete(request) → ModelResponse
        #    c. If finish_reason == "stop" → return text response
        #    d. If finish_reason == "tool_call":
        #       - Parse tool calls
        #       - For each tool call:
        #         i.   Validate schema
        #         ii.  Build McpRequest
        #         iii. gate.handle_agent_tool_call(principal, mcp_request)
        #         iv.  Audit the tool call
        #         v.   Append result to session
        #    e. If timeout or max iterations → return error
        # 3. Return final response
```

### 5.5 Tool-Call Pipeline

```
Model Output (text)
    ↓
Tool-Call Parser (extract JSON tool calls from model output)
    ↓
Schema Validation (validate arguments match tool schema)
    ↓
Capability Validation (tool exists in CATALOG, capability known)
    ↓
Policy Gate (policy.evaluate → Decision)
    ↓
Approval Gate (if needs_approval → pause, wait for approval)
    ↓
MCP Client (mcp_client.call → McpResult)
    ↓
Result Sanitization (strip secrets from result before feeding to model)
    ↓
Session (append tool result to trajectory)
    ↓
Model (next inference call with tool result in context)
```

### 5.6 Integration with Existing Gate

The agent loop does NOT bypass Gate. It uses Gate as the authority:

```python
# In Gate (core.py) — new method:
def handle_agent_tool_call(
    self,
    principal: Principal,
    server_id: str,
    tool_id: str,
    args: dict,
    request_id: str,
) -> McpResult:
    """Internal tool-call entry point for agent loop.
    
    Pipeline: catalog lookup → scope validate → policy → approval → MCP → audit.
    Same as handle_mcp but without HTTP schema validation.
    """
    spec = tool_lookup(server_id, tool_id)
    if spec is None:
        raise ValidationError(f"unknown tool: {server_id}.{tool_id}")

    resource = self._mcp_validate_scope_for_agent(server_id, tool_id, args, spec)

    decision = self.policy.evaluate(
        role=principal.role,
        command=spec.command_id,
        capability=spec.capability,
        resource=resource,
    )

    if not decision.allowed:
        self._audit_mcp_agent(server_id, tool_id, args, decision, "denied", spec)
        return McpResult(status="error", error_class="denied",
                         data={"error": f"policy denied: {decision.reason}"})

    if decision.needs_approval:
        # Agent loop handles approval flow
        return McpResult(status="error", error_class="approval_required",
                         data={"error": "approval required",
                               "request_id": request_id})

    try:
        result = self.mcp_client.call(server_id, tool_id, args)
    except Exception as e:
        self._audit_mcp_agent(server_id, tool_id, args, decision, "error", spec)
        return McpResult(status="error", error_class="mcp_error",
                         data={"error": str(e)})

    self._audit_mcp_agent(server_id, tool_id, args, decision, "executed", spec,
                          result_status=result.status)
    return result
```

---

## 6. Files To Create/Modify

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
| `gate/policy/classes.py` | Add `AGENT = "AGENT"` to Capability enum |
| `gate/core.py` | Add `handle_agent_tool_call()` method |
| `gate/config.py` | Add agent config fields (model_path, max_iterations, etc.) |
| `gate/factory.py` | Wire agent provider into Gate construction |
| `gate/ingress/schemas.py` | Add "agent" to `_SOURCES` |

### NOT Modified
| File | Reason |
|------|--------|
| `gate/cloud/*` | Cloud execution remains independent |
| `gate/mcp/*` | MCP boundary unchanged |
| `gate/audit/*` | Audit mechanism unchanged |
| `gate/approval/*` | Approval mechanism unchanged |
| `gate/heartbeat/*` | Heartbeat unchanged |

---

## 7. Test Strategy

### 7.1 Unit Tests (test_f7_agent.py)
- ModelProvider protocol compliance
- FakeModelProvider: deterministic responses
- ModelRequest/ModelResponse serialization
- Tool-call parsing (valid, malformed, missing args)
- Tool-call schema validation
- Session: message append, trajectory recording
- Trajectory: append-only, no secrets
- AgentLoop: single-turn (text response), multi-turn (tool calls), max iterations, timeout

### 7.2 Integration Tests
- Agent tool call → Gate → Policy → MCP → result
- Agent tool call denied by policy
- Agent write → approval required
- Agent tool call with invalid arguments
- Agent tool call to unknown tool
- Agent loop: provider failure recovery
- Agent loop: cancellation
- Agent loop: audit entries created

### 7.3 Security Regression Tests
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

### 7.4 Existing Test Regression
- All 304 existing tests must pass unchanged
- F7 adds tests, does not modify existing test behavior

---

## 8. Security Boundary Analysis

### What Model CAN Do
- Select a tool from available tools (CATALOG)
- Generate arguments for the selected tool
- Request tool execution via structured output
- Receive tool results in context

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

### Trust Boundary
```
UNTRUSTED (Model Output)
    ↓ validation
TRUSTED (Structured Tool Call)
    ↓ policy
AUTHORIZED (Policy Decision)
    ↓ approval (if needed)
APPROVED (User Confirmation)
    ↓ MCP
EXECUTED (Tool Result)
    ↓ sanitization
SAFE (Sanitized Result → Model)
```

### Threat Model
1. **Model hallucinates tool call** → Schema validation rejects
2. **Model generates invalid JSON** → Parser rejects, loop continues
3. **Model requests unauthorized tool** → Policy denies
4. **Model requests write without approval** → Approval gate blocks
5. **Model tries to access vault/credentials** → Policy denies (no vault rules for agent)
6. **Model generates infinite tool calls** → max_iterations stops loop
7. **Model output contains prompt injection** → Tool results sanitized before model sees them
8. **Provider fails** → Loop handles error, returns to user
9. **Tool execution fails** → Error captured, loop continues or terminates

---

## 9. Tiny Model Evaluation / Benchmark Plan

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
1. **Synthetic test suite**: 50-100 predefined scenarios with expected tool calls
2. **FakeModelProvider**: Deterministic responses for CI testing
3. **Real model evaluation**: Run against actual tiny model on vox-space
4. **Comparison baseline**: FakeModelProvider with perfect responses = 100% pass rate

### Daemon Architecture (F7-P6)
If model proves suitable:
```
uta-local-intel.service (systemd oneshot or simple)
├── Model lifecycle (load, unload, reload)
├── Request queue (inference requests)
├── Inference engine (llama-cpp-python or ONNX runtime)
├── Health check endpoint
└── IPC via Unix socket or HTTP localhost
```

Daemon constraints:
- Runs as dedicated user (not root)
- No filesystem access outside model dir
- No network access (except localhost for IPC)
- No vault access
- No credential access
- Resource limits: MemoryMax, CPUQuota

---

## 10. Recommendations

### Should tiny model/daemon start now?
**YES, but incrementally.**

1. **F7-P0 (now)**: Architecture report (this document). No code.
2. **F7-P1**: ModelProvider abstraction + FakeModelProvider. No real model needed.
3. **F7-P2**: Session/Trajectory. State management.
4. **F7-P3**: AgentLoop + tool-call pipeline with FakeModelProvider.
5. **F7-P4**: LocalModelProvider with actual tiny model. Benchmark.
6. **F7-P5**: Daemon separation if model proves suitable.
7. **F7-P6**: Integration with external clients (Telegram bot, etc.)

**Rationale**: The abstraction layer (F7-P1 through F7-P3) is model-agnostic. We can test the entire agent loop with FakeModelProvider before ever downloading a real model. This de-risks the architecture.

### What NOT to do now
- Do not download/quantize any model yet
- Do not install llama-cpp-python yet
- Do not create systemd daemon yet
- Do not integrate with Telegram bot yet
- Do not modify existing F2-F6 behavior

---

## 11. Summary

### F7-P0 Deliverables
1. ✅ This architecture report
2. ✅ Existing integration points (Section 2)
3. ✅ Reusable abstractions (Section 3)
4. ✅ Conflicts/gaps (Section 4)
5. ✅ Proposed architecture (Section 5)
6. ✅ Files to create/modify (Section 6)
7. ✅ Test strategy (Section 7)
8. ✅ Security boundary analysis (Section 8)
9. ✅ Tiny model benchmark plan (Section 9)
10. ✅ Recommendations (Section 10)

### F7-P1 Next Step
Implement `gate/agent/provider.py`:
- ModelProvider protocol
- ModelRequest, ModelResponse, ToolCall, ToolDefinition
- ModelMetadata, TokenUsage
- FakeModelProvider for testing

No real model, no daemon, no external integration.

---

## F7-P0 = COMPLETE

Awaiting review before proceeding to F7-P1.
