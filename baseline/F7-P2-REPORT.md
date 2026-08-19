# F7-P2 Report — Agent Loop + Tool-Call Orchestration

> Status: COMPLETE (corrective applied)
> Date: 2026-08-18
> Baseline: F6 304 tests, F7-P1 361 tests
> Test count: 361 -> 422 (61 new P2 tests, 0 regression)
> Corrective: F7-P2 architectural regression fixed

## A. Files Changed

### New Files

| File | Purpose | Lines |
|------|---------|-------|
| `gate/agent/session.py` | AgentSession, AgentState, TrajectoryEntry, AgentResult | ~115 |
| `gate/agent/tool_call.py` | Tool-call validation, catalog_to_tool_definitions | ~90 |
| `gate/agent/loop.py` | AgentLoop orchestrator | ~310 |
| `tests/test_f7_agent_loop.py` | 61 unit/integration tests | ~920 |

### Modified Files

| File | Change | F7-P2 Corrective |
|------|--------|-------------------|
| `gate/core.py` | Added `handle_agent_tool_call()`, `execute_approved_tool_call()`, `_agent_mcp_execute()`, `_audit_agent_tool()`, `_AgentScopeReq` | YES |
| `gate/ingress/schemas.py` | Added `"agent"` to `_SOURCES` | YES |
| `gate/agent/loop.py` | Switched from `handle_mcp()` to `handle_agent_tool_call()`, added `resume()`, `_run_loop()` | YES |
| `tests/test_heartbeat.py` | Updated `_SOURCES` assertion to include `"agent"` | YES |

## B. Architectural Regression — Fixed

### Original Problem

F7-P2 initial implementation had AgentLoop calling `Gate.handle_mcp()` instead of the designed `Gate.handle_agent_tool_call()`. The approval resume path (WAITING_APPROVAL → ApprovalStore → resume → binding verification → scope re-validation → policy re-evaluation → execute_approved_tool_call) was NOT implemented.

### What Was Fixed

1. **`Gate.handle_agent_tool_call()`** — Agent-specific method on Gate. Skips HTTP schema validation (agent loop already validated). Same pipeline: catalog → scope → policy → approval → MCP → audit. Returns `(status_code, dict)`.

2. **`Gate.execute_approved_tool_call()`** — Resume path for approved tool calls. Loads approval from store, verifies binding (request_id, command, capability, arg_hash, requester), re-validates scope using original args, re-evaluates policy, executes via MCP. Original args stored in `Gate._agent_args` dict (not on Approval dataclass which only has arg_hash).

3. **`AgentLoop._execute_tool_call()`** — Now calls `handle_agent_tool_call()` instead of `handle_mcp()`. Returns `(needs_approval, result_text)` tuple. When approval required, transitions to WAITING_APPROVAL and returns pending_request_id.

4. **`AgentLoop.resume()`** — Loads persisted session, calls `execute_approved_tool_call()`, appends result to trajectory, continues inference loop via `_run_loop()`.

5. **`AgentLoop._run_loop()`** — Extracted core inference loop shared by `run()` and `resume()`.

## C. Execution Trajectory (Corrected)

```
User Request
    |
    v
AgentLoop.run(user_message)
    |
    v
AgentSession: RUNNING
    |
    v
ModelProvider.complete(ModelRequest)
    |
    v
ModelResponse
    |
    +-- text -> COMPLETED
    |
    +-- tool_calls -> for each:
        |
        v
    validate_tool_call(tc) -> (server_id, tool_id, ToolSpec)
        |
        v
    validate_tool_call_args(tc, spec) -> args
        |
        v
    Gate.handle_agent_tool_call(principal, server_id, tool_id, args, request_id)
        |
        +-- catalog lookup -> ToolSpec
        +-- scope validation
        +-- policy evaluate (role=principal.role, capability=spec.capability)
        +-- if needs_approval:
        |       ApprovalStore.create() -> PENDING
        |       AgentState -> WAITING_APPROVAL
        |       return pending_request_id to caller
        |
        +-- if allowed:
        |       McpClient.call() -> McpResult
        |       AuditLog.append(source="agent", ...)
        |       return result
        |
        v
    ToolResult -> TrajectoryEntry(role="tool")
        |
        v
    Back to ModelProvider (next iteration)

    === After approval granted externally ===

AgentLoop.resume()
    |
    v
Gate.execute_approved_tool_call(principal, request_id)
    |
    +-- ApprovalStore.get(request_id) -> Approval
    +-- Retrieve original args from Gate._agent_args
    +-- Verify arg_hash integrity
    +-- Re-validate scope with original args
    +-- Re-evaluate policy (rules may have changed)
    +-- ApprovalStore.consume() with binding check
    +-- McpClient.call() -> McpResult
    +-- AuditLog.append(source="agent", result="executed")
    |
    v
ToolResult -> TrajectoryEntry(role="tool")
    |
    v
Back to ModelProvider (next iteration)
```

## D. AgentLoop Orchestration

AgentLoop is an ORCHESTRATOR. It:
- Calls ModelProvider for inference
- Validates tool calls against catalog
- Delegates execution to `Gate.handle_agent_tool_call()` (NOT `handle_mcp()`)
- Handles approval pausing (WAITING_APPROVAL state)
- Resumes after approval via `Gate.execute_approved_tool_call()`
- Records trajectory

AgentLoop is NOT:
- Policy engine
- Authorization system
- Approval authority
- MCP executor
- Vault
- Shell executor

## E. State Machine

```
NEW -> RUNNING -> COMPLETED
           |-> FAILED
           |-> CANCELLED
           |-> WAITING_APPROVAL -> RUNNING (via resume())
                                   |-> FAILED
                                   |-> CANCELLED
```

Terminal states: COMPLETED, FAILED, CANCELLED (no transitions out).

WAITING_APPROVAL is entered when `handle_agent_tool_call()` returns approval_required. The loop pauses, returns `AgentResult(state=WAITING_APPROVAL, pending_request_id=...)`.

resume() transitions WAITING_APPROVAL -> RUNNING, executes approved tool call, continues loop.

## F. Security Invariant

**MODEL INTELLIGENCE MAY REQUEST ACTION. MODEL IDENTITY MUST NEVER GRANT AUTHORITY. F2 GATE REMAINS THE FINAL AUTHORITY.**

- `source="agent"` is provenance-only (audit trail)
- `role=principal.role` — never hardcoded admin
- `capability=spec.capability` — from ToolSpec in CATALOG
- `handle_agent_tool_call()` runs full policy pipeline
- `execute_approved_tool_call()` re-evaluates policy on resume
- MCP remains the only execution boundary

## G. Security Tests

| # | Test | Result |
|---|------|--------|
| 1 | AgentLoop calls handle_agent_tool_call (not handle_mcp) | PASS |
| 2 | Tool calls go through Gate | PASS |
| 3 | principal.role passed to Gate (not hardcoded) | PASS |
| 4 | Vault tools excluded from model definitions | PASS |
| 5 | AgentLoop has no direct MCP/filesystem/shell access | PASS |
| 6 | Approval required → WAITING_APPROVAL state | PASS |
| 7 | Resume after approval → continued execution | PASS |
| 8 | Resume without pending → FAILED | PASS |
| 9 | Real Gate: approval → approve → execute_approved_tool_call | PASS |
| 10 | Real Gate: deny blocks execution | PASS |
| 11 | Real Gate: binding verification | PASS |
| 12 | Real Gate: policy re-evaluation on resume | PASS |
| 13 | Real Gate: audit entries created | PASS |
| 14 | Real Gate: unknown request_id → error | PASS |
| 15 | Real Gate: unknown tool → error | PASS |
| 16 | Real Gate: policy denied → error | PASS |
| 17 | Real Gate: consume before approval rejected | PASS |
| 18 | Full loop: tool call → approval → resume → complete | PASS |

## H. Test Count

| Phase | Count |
|-------|-------|
| F6 baseline | 304 |
| F7-P1 new | 57 |
| F7-P2 new (corrective) | 61 |
| **Total** | **422** |
| Regression | **0** |

### Test Breakdown (P2)

| Category | Tests |
|----------|-------|
| Session / State / Trajectory | 16 |
| Tool-call validation | 10 |
| AgentLoop orchestration (mock Gate) | 17 |
| Security boundary (mock Gate) | 5 |
| Approval/resume integration (real Gate) | 13 |
| Regression | 2 |
| **Total** | **61** |

## I. Regression Result

422 passed in 15.29s. 0 failed. 0 regression.

## J. Documentation

F7-P2 report: `docs/F7-P2-REPORT.md`

## K. F7-P3 Readiness

F7-P3 can begin. The agent loop is architecturally correct:
- AgentLoop calls `Gate.handle_agent_tool_call()` (not `handle_mcp()`)
- Approval pause/resume path fully implemented and tested
- `Gate.execute_approved_tool_call()` handles binding, scope, policy re-evaluation
- FakeModelProvider proves the full execution path
- State machine handles all lifecycle transitions (including WAITING_APPROVAL)
- Security boundaries proven with 18 security-specific tests
- 422 tests green, 0 regression
