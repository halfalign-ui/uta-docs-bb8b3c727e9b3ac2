# F6 — Heartbeat & Background Execution

**Status: COMPLETE**
**Commit: 686f8b1 + d388ff5 (WorkingDirectory fix)**
**Tests: 304/304 PASS, 0 regression**
**Deployed: 2026-08-18 on vox-space (192.168.100.9)**

## Ringkasan

F6 menambahkan infrastruktur heartbeat deterministik yang dijadwalkan oleh systemd timer. AI tidak terlibat dalam scheduling. Semua task dijalankan sebagai fungsi murni dengan retry terbatas, audit, dan policy check.

## Implementasi

### Components
- `gate/heartbeat/runner.py` — HeartbeatRunner: retry (max 3), exponential backoff, policy check, audit
- `gate/heartbeat/tasks.py` — 4 tasks: health_check, cleanup_runtime, quota_report, audit_verify
- `gate/heartbeat/cli.py` — CLI entry: `python -m gate.heartbeat --task <name>`
- `gate/heartbeat/__main__.py` — Enables `python -m gate.heartbeat`

### Config Changes
- `gate/config.py` — +6 fields: heartbeat_enabled, heartbeat_interval_seconds, heartbeat_max_retries, heartbeat_runtime_dir, heartbeat_quota_path, heartbeat_log_dir

### Policy Changes
- `gate/policy/engine.py` — +`build_heartbeat_rules()`, +`_HEARTBEAT_TASKS` list (4 tasks)
- Capability: BACKGROUND (new)

### Ingress Changes
- `gate/ingress/schemas.py` — +"background" to `_SOURCES`

### Systemd Units
- `uta-heartbeat.timer` — OnBootSec=60s, OnUnitActiveSec=5min, RandomizedDelaySec=30s
- `uta-heartbeat.service` — Type=oneshot, WorkingDirectory=/srv/dev/UTA/gate

### Tests
- `tests/test_heartbeat.py` — 40 tests covering runner, tasks, policy, CLI, config, ingress

## Deployment Verification

### Systemd Status (2026-08-18)
- `uta-heartbeat.timer`: enabled, active (waiting)
- Schedule: 5-minute interval with 30s random delay
- First manual trigger: OK, health_check passed

### Audit
- source=background entries confirmed
- Hash chain integrity: OK
- No secrets in audit log: VERIFIED

### Functional Verification
- All 4 tasks execute successfully via sudo
- Timer fires on schedule
- Retry logic verified (tests)
- Policy denial for non-admin role verified (tests)
- Unknown task returns exit code 1 (tests)

## Security Regression

- deny-by-default F2: INTACT (36 tests in test_pipeline.py, test_mcp.py)
- MCP boundary F3: INTACT (25 tests in test_mcp.py)
- Vault boundary F4: INTACT (19 tests in test_vault.py)
- Cloud execution F5A: INTACT (55 tests in test_f5a.py)
- Heartbeat privilege = interactive, never more
- Heartbeat does not bypass policy/approval
- No credentials in runtime/log
- No arbitrary command execution
- No shell access

## Catatan

- Service runs as root (same as interactive SSH user)
- Audit dir owned by root (created by systemd service)
- `WorkingDirectory=/srv/dev/UTA/gate` required for module resolution
- Heartbeat task list is hardcoded, not configurable at runtime

## F6 = CLOSED
