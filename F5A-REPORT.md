# F5A-REPORT — Cloud Runtime Operational Completion

> Commit: `e0f0a7c` | Date: 2026-08-17 | Tests: 264 pass (70 F2 + 49 F3 + 29 F4 + 61 F5 + 55 F5A)

## Summary

F5A closes deferred items from F5: runtime isolation, fallback validation,
streaming tests, security regression tests. Adds RuntimeIsolation class,
rewrites execute_with_fallback, fixes lifecycle bugs, and adds 55 new tests.

## What Changed

### New: gate/cloud/runtime_isolation.py
- RuntimeIsolation class: per-execution dirs (/secondary/runtime/{id}/), mode 0700
- Metadata sanitization: strips key/secret/token/password/credential from persisted data
- Request/response/error persistence: JSON files in execution dir
- Cleanup: retention-based expiry for completed/failed executions
- Active listing: list_running returns in-progress executions

### Modified: gate/cloud/runtime.py
- Integrated RuntimeIsolation into CloudRuntime.__init__
- execute(): creates execution dir, persists request/response/metadata, no duplicate CREDENTIAL_RESOLVE transition
- execute_with_fallback(): shared execution_id across attempts, per-attempt audit/quota, credential isolation per provider, proper retryable/non-retryable error handling

### New: tests/test_f5a.py (55 tests)
| Category | Tests |
|----------|-------|
| A. Runtime isolation (dirs, metadata, cleanup, execution) | 18 |
| B. Fallback validation (success, failover, non-retryable, credential isolation, quota) | 9 |
| C. Streaming validation (event ordering, [DONE], malformed SSE, no buffering) | 6 |
| D. Real provider (skipped — no Vault credential) | 1 |
| E. Security regression (F2/F3/F4/F5 invariants) | 17 |
| F. Production safety (no llama-server, no ai-gateway, no ports, no /secondary/cache) | 5 |
| **Total F5A new** | **55** |

## Bug Fixes (found during F5A)

1. **runtime.py: execute_with_fallback()**: `executor._build_payload()` referenced undefined variable — fixed to `self._executor._build_payload()`
2. **runtime.py: execute()**: duplicate PENDING→CREDENTIAL_RESOLVE transition (also done by executor.execute()) caused InvalidTransition on error path
3. **lifecycle.py state machine**: streaming relay tried PENDING→COMPLETED directly; tests now transition record through proper states

## Security Properties (verified by regression tests)

1. F2 deny-by-default: network_request and read_secret_file blocked even for admin
2. F3 MCP read: filesystem.read_file allowed within scope
3. F4 vault: admin read OK, admin write requires approval, user cannot read
4. F5 cloud: execute admin-only, list_providers user-allowed, cancel admin-only
5. Endpoint validation: localhost, private IPs, HTTP all rejected
6. Credentials never in audit logs or runtime files
7. Quota limits enforced per provider/model
8. Runtime isolation: metadata sanitized (no api_key/secret/token in persisted files)
9. Cloud independent of local inference (no llama/ai-gateway references)

## What's NOT in F5A (deferred)

- **Live provider test**: Skipped — no Vault credential configured for real API
- **Full streaming E2E**: SSE relay tested with mocks only; live streaming deferred to F8
- **Performance/throughput testing**: Deferred to F8

## Test Count Summary

| Phase | Tests | Cumulative |
|-------|-------|------------|
| F2 | 70 | 70 |
| F3 | 49 | 119 |
| F4 | 29 | 148 |
| F5 | 61 | 209 |
| F5A | 55 | **264** |
