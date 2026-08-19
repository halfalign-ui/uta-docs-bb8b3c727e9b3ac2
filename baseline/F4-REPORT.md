# F4 Report — Vault & Secret Handling

**Status:** COMPLETE
**Date:** 2026-08-17
**Tests:** 148 pass, 0 fail (30 new F4 tests)
**Commit:** `3f5abad`

---

## What Was Built

### Vault MCP Server (`gate/mcp/servers/vault_server.py`)
- 5 tools: `list_secrets`, `read_secret`, `write_secret`, `delete_secret`, `secret_exists`
- Logical secret IDs (e.g. `cloud/openrouter/api_key`) map to `/data/vault/{path}.secret`
- No shell, no subprocess, no network. Secrets stay in memory only.
- File permissions: 0600 (vault user only)
- Read operations mark content as `sensitive: true` in response

### Vault Security (`gate/mcp/vault_security.py`)
- `validate_secret_id()`: regex validation, depth limit (5), path traversal blocking
- `secret_id_from_path()`: reverse mapping for audit use
- Physical path containment: verified that resolved path stays within vault dir

### Policy Rules
- **Admin READ on vault:** allowed (e.g. `list_secrets`, `read_secret`, `secret_exists`)
- **Admin WRITE on vault:** requires approval (`write_secret`, `delete_secret`)
- **User:** NO vault rules → denied by default
- VAULT sentinel in policy engine for resource matching

### Catalog Updates
- `ScopeKind.VAULT` added
- 5 vault tools registered (16 total catalog entries)

### Audit Safety
- `redact()` regex updated to catch `content` key (prevents secret leakage in audit logs)
- Vault args (`secret_id`) are logged; vault content values are never logged
- MCP audit logs `result_status`, not `result.data` (secret content never hits audit)

### Integration
- MCP client: vault server module mapping, `vault_dir` parameter
- Config: `vault_dir` field (default `/data/vault`)
- Factory: `vault_dir` wired to `McpClient`
- Gate core: vault scope validation in `_mcp_validate_scope()`

---

## Infrastructure (vox-space)
- System user `vault` created (no shell, no sudo, no docker)
- `/data/vault/` created, owned `vault:vault`, mode 0700
- `/etc/ai-gateway/gateway.env` remains untouched (0600, root-owned)

---

## Security Properties Preserved

| Property | Status |
|----------|--------|
| Deny-by-default | User gets NO vault rules → denied |
| No shell escape | Vault server uses only `os.path` operations |
| Path containment | `validate_secret_id` verifies physical path stays in vault |
| Approval gating | Write/delete require approval |
| Audit integrity | Hash chain intact, secret content redacted |
| Regression | All 118 F2/F3 tests still pass |

---

## Test Coverage (30 new tests)

| Category | Tests | Notes |
|----------|-------|-------|
| Path validation | 8 | traversal, null byte, depth, format |
| Roundtrip | 2 | validate → recover ID |
| Path containment | 2 | dangerous IDs, valid IDs stay in vault |
| Catalog | 4 | ScopeKind, tool count, capabilities |
| Policy | 4 | admin read/write, user deny |
| Audit redaction | 5 | content key, nested, audit log, write content |
| Regression | 3 | engine, catalog, audit chain |

---

## Migration Candidate

`/etc/ai-gateway/gateway.env` contains `AI_GATEWAY_API_KEY`. Could be migrated to vault as `cloud/ai-gateway/api_key` — but only with explicit user approval. **Not done in this phase.**

---

## What's Next

F4 is the final gate core feature. The gate now has:
- **F0:** Discovery (DONE)
- **F1:** Architecture spec (DONE)
- **F2:** Gate core (DONE)
- **F3:** MCP tool boundary (DONE)
- **F4:** Vault & secret handling (DONE)

**F5 recommendation:** Operational hardening — systemd units, log rotation, monitoring, backup/restore for vault. This is ops work, not gate code.
