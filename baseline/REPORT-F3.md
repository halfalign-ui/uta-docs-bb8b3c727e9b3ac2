# F3 — MCP TOOL BOUNDARY (Report)

> Status: IMPLEMENTED. Lokasi kode: `/srv/dev/UTA/gate/` (repo UTA).
> Referensi desain: `UTA-PLAN.md` §F3.
> Tanggal: 2026-08-17.

## Ringkasan

Gate sekarang berfungsi sebagai MCP (Model Context Protocol) client yang
berkomunikasi dengan 3 MCP server subprocess: `filesystem` (read-only),
`workspace` (read + approval-gated write), `system` (informational only).

Pipeline F3 melewati pipeline deterministik yang sama dengan F2:

```
validate → catalog lookup → scope validate → policy evaluate
        → approval (if needed) → MCP call → audit → respond
```

## Yang Dibangun

### MCP Server (3 subprocess, stdio transport)

| Server | Trust Domain | Tools | Capability |
|---|---|---|---|
| `filesystem_server.py` | filesystem | read_file, list_directory, stat | READ only |
| `workspace_server.py` | workspace | read, list, stat, write_file | READ + WRITE (approval) |
| `system_server.py` | system | system_info, service_health, disk_usage, process_summary | READ only |

Semua server menggunakan MCP SDK 2.0.0 official (`mcp==2.0.0`) dengan
konstructor callbacks (`on_list_tools`, `on_call_tool`), bukan
`add_request_handler`. Server berjalan sebagai subprocess, berkomunikasi
via stdin/stdout (stdio transport).

### MCP Client (`gate/mcp/client.py`)

- Per-call stdio connection (tidak persistent daemon).
- Timeout configurable (default 10s).
- Result size limit (default 64KB).
- Fail-closed: semua error → `McpUnavailableError` atau `McpToolError`.

### MCP Catalog (`gate/mcp/catalog.py`)

- 11 tools terdaftar sebagai single source of truth.
- ScopeKind: `PATH` (filesystem/workspace) atau `SYSTEM`.
- Lookup via `tool_lookup(server_id, tool_id)`.

### MCP Security (`gate/mcp/security.py`)

- `validate_path()`: null bytes, hidden components, absolute required,
  roots containment check (realpath-based, symlink-safe).
- Defense-in-depth: server juga validasi path; gate juga validasi.

### Policy Integration (`gate/policy/engine.py`)

- `build_mcp_rules()`: generate deny-by-default rules untuk semua
  MCP tools di catalog.
- Filesystem tools: `resource=None` (path validated by scope, not rule).
- Workspace tools: `resource=WORKSPACE` sentinel (matched against workspace_dir).
- System tools: `resource=SYSTEM` sentinel.
- Admin: READ on all tools, WRITE on workspace.write_file requires approval.
- User: READ-only.
- SECRET/NETWORK/BACKGROUND: no rule → denied.

### Audit Extension (`gate/audit/audit.py`)

- Fields baru: `server_id`, `tool_id`, `error_class`, `approval_state`, `duration_ms`.
- F3 requests logged via `_audit_mcp()`.

### Ingress (`gate/ingress/schemas.py`, `gate/ingress/server.py`)

- `McpRequest` dataclass dengan `validate_mcp_request()`.
- `POST /mcp` endpoint (auth required, payload limit).

### Gate Core (`gate/core.py`)

- `handle_mcp()`: full pipeline (validate → catalog → scope → policy → approval → execute → audit).
- `_mcp_validate_scope()`: validates path against filesystem roots or workspace dir.
- `_mcp_approval_flow()`: approval for MCP write operations, binding via `{server_id, tool_id, path}`.
- `_execute_from_context()`: routes MCP commands through `mcp_client`, F2 commands through runner.

## File Layout

```
gate/mcp/
  __init__.py
  catalog.py          # 11 tools, ToolSpec, ScopeKind, lookup()
  client.py           # McpClient (stdio per-call, timeout, size limit)
  security.py         # validate_path(), validate_path_exists()
  servers/
    __init__.py
    filesystem_server.py   # read-only MCP server
    workspace_server.py    # workspace MCP server (read + write)
    system_server.py       # system info MCP server
```

## Verifikasi

- **118 test** (F2: 70 + F3: 48) — `python3 -m unittest discover -s tests -t .` → **OK** (3 skipped: docker.sock, /secondary, /data/vault not present).
- Cakupan F3:
  - Client connects to real MCP server subprocess (system_info).
  - Tool discovery (11 tools in catalog).
  - Filesystem read within scope → ok.
  - Filesystem path outside scope → McpResourceError.
  - `../` traversal → McpResourceError.
  - Symlink escape → McpResourceError.
  - Workspace read → ok.
  - Workspace write requires approval → approval_required.
  - Workspace write bypass denied.
  - System info → ok.
  - Unknown tool rejected (ValidationError).
  - SECRET capability → PermissionDenied (policy deny-by-default).
  - Docker socket → McpResourceError.
  - Hidden paths (.env, .ssh) → McpResourceError.
  - Oversized result → server rejects.
  - Non-JSON response handled gracefully.
  - Timeout on bad server → McpUnavailableError.
  - Unknown server → McpUnavailableError.
  - Policy denial audited.
  - Successful call audited.
  - Approval → approve → execute (write_file) → file written.
  - Token not in audit log.
  - F2 commands still work alongside F3.
  - Security: shell syntax, semicolon, pipe, /etc/shadow, SSH keys, home dir, null byte, relative path, hidden git dir → all denied.

## Keputusan Desain

1. **MCP SDK 2.0.0**: Constructor callbacks (`on_list_tools`, `on_call_tool`)
   bukan `add_request_handler`. Handler signature: `async (ctx, params) -> result`.
   Params type: `CallToolRequestParams` (`.name`, `.arguments`), bukan
   `CallToolRequest` (yang punya `.params.name`).

2. **Per-call stdio**: Setiap `client.call()` spawn subprocess baru, kirim
   JSON-RPC, baca response, kill. Lebih lambat tapi lebih aman (tidak ada
   persistent daemon yang bisa crash/stale).

3. **Scope validation di Gate**: Gate validates path containment sebagai
   defense-in-depth. Server juga validate, tapi Gate tidak bergantung pada
   server validation.

4. **Filesystem resource=None**: Karena path validation sudah dilakukan
   di `_mcp_validate_scope()`, policy rules untuk filesystem tools tidak
   perlu lagi cek resource containment.

5. **Approval binding**: MCP write_file approval di-bind ke
   `{server_id, tool_id, path}` (tidak termasuk content) untuk stabilitas.
   Binding hash disimpan di context untuk consume.

## Keamanan

- MCP bukan security boundary — F2 Gate tetap ultimate auth boundary.
- Deny-by-default di semua lapisan.
- Tool catalog whitelist: hanya 11 tools yang terdaftar.
- Path validation: realpath, no symlink escape, no hidden components, no null bytes.
- No shell, no sudo, no Docker socket, no vault/cloud/Telegram/LLM.
- Approval flow identik dengan F2 (state machine, binding, TTL, non-reusable).
- All requests audited dengan server_id, tool_id, duration_ms.

## Status

```
F3: COMPLETE
SYSTEM CHANGES: NONE (hanya file baru di repo)
INSTALLATIONS: mcp==2.0.0 di gate/.venv (Python 3.13)
PRODUCTION SERVICES: UNCHANGED
```

## Next

F4 — Vault & Secret Handling.
