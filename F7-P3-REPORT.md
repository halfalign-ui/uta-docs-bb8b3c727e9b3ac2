# F7-P3: Local Intelligence Runtime — Complete Report

**Status**: COMPLETE  
**Commit**: pending  
**Test count**: 457 (35 new, 422 existing, 0 regression)  
**Infrastructure**: llama.cpp (CUDA) + Qwen2.5-7B-Instruct-Q4_K_M on vox-space (RTX 4060 8GB)

---

## What Was Built

P3 implements the **local intelligence layer** — a privacy-preserving, policy-constrained local LLM that serves as the third and final intelligence tier alongside Cloud (F1) and Background (F1B).

### LocalModelProvider (`gate/agent/providers/local.py`)

A `ModelProvider` implementation that communicates with a local llama-server instance via OpenAI-compatible HTTP API.

**Architecture**: Model → llama.cpp (HTTP) → OpenAI-compatible API → LocalModelProvider → AgentLoop → Gate

**Key properties**:
- **No new dependencies**: stdlib `urllib.request` only — no httpx, no aiohttp
- **No security bypass**: Cannot execute tools, approve calls, access Vault, or modify Principal
- **Tool-calling support**: Full conversion between UTA `ToolDefinition`/`ToolCall` and OpenAI format
- **Failure modes**: Returns structured errors, never executes silently
- **Request mapping**: `ModelRequest` → OpenAI payload (`messages`, `tools`, `tool_choice`, `max_tokens`, `temperature`)
- **Response parsing**: Handles text, tool calls, multi-turn, malformed arguments, missing fields

### Security Invariants

1. **Model is intelligence, not authority**: Provider has no `approve()`, `execute()`, `run_tool()`, `call_mcp()`, `shell`, `vault`, `filesystem`, `principal`, or `set_principal` attributes
2. **Tool definitions carry no authorization metadata**: No `capability`, `approval`, `policy`, `secret_token`, `vault_token`, or `api_key` in the payload sent to the model
3. **No credentials in requests**: No GitHub tokens (`gho_`), passwords, API keys, or secrets in the model payload
4. **Standard ModelProvider contract**: Uses same `complete()` / `is_available()` / `metadata()` as FakeModelProvider — no special privileges

### Infrastructure

| Component | Value |
|-----------|-------|
| Server | llama.cpp (CUDA, built from source) |
| Model | Qwen2.5-7B-Instruct-Q4_K_M (4.4GB) |
| GPU | RTX 4060 (8GB VRAM) |
| Endpoint | `http://127.0.0.1:8080/v1/` (OpenAI-compatible) |
| Systemd | `llama-server.service` |
| API | `/v1/chat/completions` (supports tools, multi-turn) |

### Test Categories

| Category | Count | Description |
|----------|-------|-------------|
| Unit: Metadata | 2 | Provider ID, caching |
| Unit: Request mapping | 6 | Payload construction, tool definitions, max_tokens |
| Unit: Response parsing | 8 | Text, tool calls, malformed, edge cases |
| Security | 10 | No bypass, no secrets, no credentials, error types |
| Integration | 9 | Real llama-server: text, tools, multi-turn, AgentLoop |

---

## Security Posture

```
+-----------------------------------------------------------+
|                    POLICY LAYER (F2)                       |
|    Always runs. Authority boundary.                        |
|    Cannot be bypassed by anything in this file.            |
+-----------------------------------------------------------+
|                  TOOL EXECUTION (F2)                       |
|    MCP only. One method. One path.                         |
+-----------------------------------------------------------+
|                 INTELLIGENCE LAYER (F7)                     |
|    LocalModelProvider: pure inference only                 |
|    Cannot: execute, approve, access vault, change role    |
|    No new dependencies, no new access paths               |
+-----------------------------------------------------------+
```

---

## Next Steps

P3 is now **COMPLETE**. The full F7 Local Intelligence Runtime is operational:

- **P0**: Architecture + Security Preflight ✅
- **P1**: ModelProvider Contract + FakeModelProvider ✅
- **P2**: Agent Loop Integration + Approval Flow ✅
- **P3**: Local Runtime (local intelligence layer) ✅

The LocalModelProvider provides a real local inference backend for the agent loop. Tool calling works end-to-end: Model → llama-server → LocalModelProvider → AgentLoop → Gate → MCP → policy check → execute → resume.
