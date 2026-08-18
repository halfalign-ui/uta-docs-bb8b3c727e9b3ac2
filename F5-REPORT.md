# F5-REPORT — Cloud Execution & Agent Runtime

> Commit: `4ad5e05` → F5A update `e0f0a7c` | Date: 2026-08-17 | Tests: 264 pass (was 209 at F5, +55 F5A)

## Summary

F5 menambahkan cloud execution boundary ke UTA Gate. Cloud inference
dilakukan melalui controlled, auditable path — provider config di JSON,
credential resolved via F4 Vault MCP, quota enforcement, lifecycle
state machine, dan 5 MCP tools baru.

F5A menutup item deferred: runtime isolation, fallback validation,
streaming tests, security regression.

## Architecture

```
Caller → Gate.handle_mcp("cloud.execute", ...)
  → Policy: admin only, NETWORK capability
  → CloudRuntime.execute()
    → RuntimeIsolation.create_execution_dir()
    → CredentialResolver.resolve(provider_id) → Vault MCP
    → QuotaTracker.check(provider_id, model_id)
    → CloudExecutor._call(provider, credential, messages) → HTTPS
    → QuotaTracker.record(...)
    → ExecutionStore.put(record)
    → RuntimeIsolation.persist_response/metadata()
  → Audit trail
```

## Files Added/Modified

### gate/cloud/ (13 files)
| File | Purpose |
|------|---------|
| `__init__.py` | Package init |
| `errors.py` | Domain exceptions |
| `provider.py` | Dataclasses: ProviderConfig, ModelSpec, PricingInfo |
| `config.py` | CloudConfig, QuotaLimits |
| `registry.py` | ProviderRegistry + endpoint validation |
| `credential.py` | CredentialResolver (via Vault MCP) |
| `lifecycle.py` | ExecState state machine + ExecutionRecord |
| `quota.py` | QuotaTracker |
| `executor.py` | CloudExecutor (urllib, SSL mandatory) |
| `streaming.py` | StreamingRelay (SSE provider→caller) |
| `store.py` | ExecutionStore (JSONL persistence) |
| `runtime.py` | CloudRuntime (orchestrator, fallback, cancel) — **F5A: +RuntimeIsolation** |
| `runtime_isolation.py` | **F5A: RuntimeIsolation** (dirs, sanitization, cleanup) |

### gate/mcp/servers/cloud_server.py
5 MCP tools: execute, list_providers, execution_status, quota_status, cancel.

### Tests
| File | Tests |
|------|-------|
| `test_cloud.py` | 61 (F5) |
| `test_f5a.py` | 55 (F5A) |
| **Total cloud** | **116** |

## Security Properties

1. HTTPS-only: HTTP, private IPs, localhost blocked.
2. No credential logging: API keys resolved at runtime, never in audit.
3. Deny-by-default: cloud.execute admin-only.
4. Quota enforcement: per-provider, per-model limits.
5. Audit trail: every execution attempt logged.
6. Runtime isolation: per-execution dirs, sanitized metadata.
7. Credential isolation: separate API keys per fallback attempt.
8. Cloud independent: no references to llama-server or ai-gateway.

## Execution Lifecycle

```
PENDING → CREDENTIAL_RESOLVE → CONNECTING → RUNNING → COMPLETED
                   ↓                ↓           ↓
                 FAILED          FAILED      TIMEOUT
                                    ↓
                                CANCELLED
```

## Bug Fixes (F5 + F5A)

1. lifecycle.py: _VALID_TRANSITIONS used bare names → NameError
2. registry.py: 172.16-31.x validation caught itself
3. test_mcp.py: hardcoded catalog count
4. test_cloud.py: catalog count off-by-one
5. **F5A**: runtime.py execute_with_fallback() undefined executor variable
6. **F5A**: runtime.py execute() duplicate CREDENTIAL_RESOLVE transition
7. **F5A**: Streaming test records not in correct state for COMPLETED transition

## What Changed from F5 to F5A

- `runtime.py`: +RuntimeIsolation integration, fixed fallback, removed duplicate transition
- `runtime_isolation.py`: NEW — dirs, sanitization, cleanup
- `test_f5a.py`: NEW — 55 tests covering isolation, fallback, streaming, security, production
- **Total: 209 → 264 tests (+55 F5A)**
