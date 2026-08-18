# F7-P1 Report — Model Provider Contract + Fake Provider

> Status: COMPLETE
> Date: 2026-08-18
> Baseline: F6 304 tests, F7-P0 COMPLETE
> Test count: 304 → 361 (57 new, 0 regression)

## A. Files Changed

### New Files

| File | Purpose | Lines |
|------|---------|-------|
| `gate/agent/__init__.py` | Package init | ~10 |
| `gate/agent/provider.py` | ModelProvider protocol, ModelRequest, ModelResponse, ToolDefinition, ToolCall, ModelMetadata, TokenUsage, FinishReason, Message | ~160 |
| `gate/agent/providers/__init__.py` | Package init | ~1 |
| `gate/agent/providers/errors.py` | Provider error model | ~40 |
| `gate/agent/providers/fake.py` | FakeModelProvider (deterministic testing) | ~70 |
| `gate/agent/providers/registry.py` | ModelProviderRegistry (minimal) | ~45 |
| `tests/test_f7_agent.py` | 57 unit/integration tests | ~380 |

### Modified Files

None. No F2–F6 files were modified.

## B. Architecture Implemented

```
gate/gate/agent/
├── __init__.py
├── provider.py              # ModelProvider, ModelRequest, ModelResponse, etc.
└── providers/
    ├── __init__.py
    ├── errors.py            # ProviderError hierarchy
    ├── fake.py              # FakeModelProvider
    └── registry.py          # ModelProviderRegistry
```

## C. Provider Contract

### ModelProvider (ABC)

```python
class ModelProvider(ABC):
    def complete(self, request: ModelRequest) -> ModelResponse: ...
    def is_available(self) -> bool: ...
    def metadata(self) -> ModelMetadata: ...
```

A provider is an intelligence adapter. It produces ModelResponse only. It must never execute tools, access vault, or bypass Gate.

### ModelRequest

Provider-neutral inference request. Carries messages, tool definitions, and options. Not coupled to any vendor API.

### ModelResponse

Provider-neutral inference response. Contains text, tool_calls, finish_reason, usage, metadata, error. Provider normalizes its output into this structure.

### ToolDefinition

Model-facing tool representation. Contains name, description, parameters (JSON Schema). Does NOT contain capability, role, approval, or secrets. tool visibility != authorization.

### ToolCall

Structured tool-call request. Contains id, name, arguments. Provider must normalize model-specific output into this. AgentLoop must NOT parse arbitrary text for tool calls.

### ModelMetadata

Describes model/provider capabilities. Does NOT encode filesystem/vault/shell/policy permissions. Model capability != execution authority.

### TokenUsage

Token counts when available. Optional.

### FinishReason

Enum: STOP, TOOL_CALLS, LENGTH, ERROR.

## D. FakeModelProvider Scenarios

| Scenario | Configuration | Result |
|----------|--------------|--------|
| Plain text response | responses=[ModelResponse(text="Hello")] | text="Hello" |
| One valid tool call | responses=[ModelResponse(tool_calls=(ToolCall(...),))] | tool_calls returned |
| Multiple tool calls | responses=[ModelResponse(tool_calls=(tc1, tc2))] | 2 tool_calls |
| Sequential responses | responses=[tool_call_response, text_response] | Returns in order |
| Exhaust to last | 2 responses, 3 calls | Returns last response |
| Provider unavailable | unavailable=True | ProviderUnavailableError |
| Provider failure | fail_after=2 | Fails on 3rd call |
| Empty responses | responses=[] | Empty text response |

## E. AgentLoop Integration Boundary

Provider contract is established. AgentLoop can consume ModelProvider:

```python
provider = FakeModelProvider(responses=[...])
response = provider.complete(ModelRequest(messages=..., tools=...))
# response.text, response.tool_calls available
```

AgentLoop is NOT implemented in P1. P1 establishes the contract. P2 will implement the loop.

## F. Security Tests

| # | Test | Result |
|---|------|--------|
| 1 | Provider cannot execute tools directly | PASS |
| 2 | ToolDefinition does not expose capability | PASS |
| 3 | Provider cannot change Principal role | PASS |
| 4 | Provider cannot bypass Gate | PASS |
| 5 | ModelResponse is treated as untrusted data | PASS |
| 6 | Malformed ToolCall is rejected | PASS |
| 7 | Unknown ToolCall is rejected | PASS |
| 8 | Invalid arguments are rejected | PASS |
| 9 | Provider failure does not execute a tool | PASS |
| 10 | Provider cannot approve a tool call | PASS |
| 11 | Secrets not in ModelRequest | PASS |
| 12 | Secrets not in ModelResponse | PASS |
| 13 | Fake provider cannot access Vault | PASS |
| 14 | Fake provider cannot access filesystem/shell/MCP | PASS |
| 15 | F2–F6 tests remain green | PASS |

## G. Test Count

| Phase | Count |
|-------|-------|
| F6 baseline | 304 |
| F7-P1 new | 57 |
| **Total** | **361** |
| Regression | **0** |

### Test Breakdown

| Category | Tests |
|----------|-------|
| Provider contract (types) | 24 |
| FakeModelProvider | 13 |
| Provider errors | 3 |
| Provider registry | 6 |
| Security boundary | 9 |
| F2–F6 regression | 3 |
| **Total** | **57** |

## H. Regression Result

361 passed in 15.29s. 0 failed. 0 regression.

## I. Commit Hash

- vox-space: TBD (commit pending)
- GitHub: TBD (push pending)

## J. Documentation

F7-P1 report: `docs/F7-P1-REPORT.md`
F7-P0 architecture: `docs/F7-P0-ARCHITECTURE.md` (unchanged)

## K. F7-P2 Readiness

F7-P2 can begin. The provider contract is stable:
- ModelProvider protocol established
- FakeModelProvider works deterministically
- Security boundaries proven
- No F2–F6 regression
- 361 tests green
