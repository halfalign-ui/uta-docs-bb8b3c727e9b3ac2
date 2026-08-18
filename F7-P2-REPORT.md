# F7-P2 Report — Agent Loop + Tool-Call Orchestration

> Status: COMPLETE
> Date: 2026-08-18
> Baseline: F6 304 tests, F7-P1 361 tests
> Test count: 361 -> 409 (48 new, 0 regression)

## A. Files Changed

### New Files

| File | Purpose | Lines |
|------|---------|-------|
| `gate/agent/session.py` | AgentSession, AgentState, TrajectoryEntry, AgentResult | ~115 |
| `gate/agent/tool_call.py` | Tool-call validation, catalog_to_tool_definitions | ~90 |
| `gate/agent/loop.py` | AgentLoop orchestrator | ~195 |
| `tests/test_f7_agent_loop.py` | 48 unit/integration tests | ~380 |
| `docs/F7-P2-REPORT.md` | Report | ~130 |

### Modified Files

None. No F2-F6 files were modified.

## B. Architecture Implemented

```
gate/gate/agent/
├── __init__.py
├── provider.py              # P1: ModelProvider contract
├── providers/               # P1: FakeModelProvider, errors, registry
├── session.py               # P2: AgentSession, AgentState, Trajectory
├── tool_call.py             # P2: tool-call validation, catalog -> ToolDefinition
└── loop.py                  # P2: AgentLoop orchestrator
```

## C. Execution Trajectory (Proven)

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
    Gate.handle_mcp(principal, body)  <-- F2 pipeline
        |
        +-- policy evaluate
        +-- approval (if needed)
        +-- MCP call
        +-- audit
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
- Delegates execution to Gate.handle_mcp()
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
           |-> WAITING_APPROVAL -> RUNNING
                                   |-> FAILED
                                   |-> CANCELLED
```

Terminal states: COMPLETED, FAILED, CANCELLED (no transitions out).

## F. Security Tests

| # | Test | Result |
|---|------|--------|
| 1 | source=agent in all Gate calls | PASS |
| 2 | Tool calls go through Gate.handle_mcp() | PASS |
| 3 | principal.role passed to Gate (not hardcoded) | PASS |
| 4 | Vault tools excluded from model definitions | PASS |
| 5 | AgentLoop has no direct MCP/filesystem/shell access | PASS |

## G. Test Count

| Phase | Count |
|-------|-------|
| F6 baseline | 304 |
| F7-P1 new | 57 |
| F7-P2 new | 48 |
| **Total** | **409** |
| Regression | **0** |

### Test Breakdown (P2)

| Category | Tests |
|----------|-------|
| Session / State / Trajectory | 16 |
| Tool-call validation | 10 |
| AgentLoop orchestration | 16 |
| Security boundary | 5 |
| Regression | 2 |
| **Total** | **48** |

## H. Regression Result

409 passed in 15.30s. 0 failed. 0 regression.

## I. Commit Hash

- vox-space: TBD (commit pending)
- GitHub: TBD (push pending)

## J. Documentation

F7-P2 report: `docs/F7-P2-REPORT.md`

## K. F7-P3 Readiness

F7-P3 can begin. The agent loop is stable:
- AgentLoop orchestrates provider -> tool calls -> Gate -> trajectory
- FakeModelProvider proves the full execution path
- State machine handles all lifecycle transitions
- Security boundaries proven
- 409 tests green
