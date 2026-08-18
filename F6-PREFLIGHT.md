# F6 PREFLIGHT — Heartbeat & Background Execution

> Date: 2026-08-17 | Status: DESIGN ONLY — AWAITING APPROVAL

## 1. Current State (Verified)

### Repository Structure
- 47 Python source files across 10 subpackages (gate/)
- 12 test files, 264 tests, all passing
- Key entry: gate/cli.py -> gate/core.py (Gate class)

### Existing Systemd
- **No UTA systemd units exist** -- no service, no timer
- Only: ai-gateway.service, llama-server.service (local inference, not UTA)
- 19 system timers (apt, sysstat, logrotate, etc.) -- none UTA-related
- **Confirmed: no heartbeat/scheduler exists anywhere in UTA**

### /secondary Layout
```
/secondary/
  backups/    cache/ (benchmark data -- DO NOT TOUCH)
  logs/       lost+found/
  runtime/    (created by F5A RuntimeIsolation, per-execution dirs)
```

### Capability Enum (policy/classes.py)
- Includes BACKGROUND capability but NO policy rules grant it
- Result: BACKGROUND is deny-by-default (correct, must add rules)

### Audit System
- Append-only JSONL with SHA-256 hash chain
- Fields: source, request_id, command, capability, decision, args, etc.
- Already supports arbitrary source values (currently: dev, cloud-l2, worker)
- Background source needs to be added to _SOURCES set in ingress/schemas.py

### Policy Engine
- Deny-by-default, first-match decides
- No BACKGROUND rules exist -> all background requests denied
- Must add: background.execute and background.cleanup allow rules for admin only

### Event Bus
- In-process pub/sub for SSE streaming
- Background results could be published here for monitoring

## 2. F6 Design

### 2.1 Architecture

```
systemd timer (uta-heartbeat.timer)
  -> triggers on configurable interval (default: 5min)
  -> runs uta-heartbeat.service (Type=oneshot)
    -> python -m gate.heartbeat --task <task_type>
      -> HeartbeatRunner:
        1. Generate unique request_id (bg_{timestamp}_{random})
        2. Set source="background"
        3. Evaluate policy (BACKGROUND capability)
        4. Execute bounded task
        5. Record outcome in audit log
        6. Retry on failure (max 3, exponential backoff)
        7. Exit 0 on success, exit 1 on failure
```

### 2.2 Task Types

| Task | Description | Bounded |
|------|-------------|---------|
| health_check | Verify gate is responsive, audit chain intact | 30s |
| cleanup_runtime | Remove expired execution dirs from /secondary/runtime | 60s |
| quota_report | Aggregate cloud quota usage, write summary | 30s |
| audit_verify | Verify audit hash chain integrity | 30s |

### 2.3 HeartbeatRunner (gate/heartbeat/runner.py)

Core class that executes background tasks:

- __init__(config, audit, policy)
- run(task_name) -> exit code
- Retry loop: max 3 attempts, exponential backoff (2s, 4s, 8s)
- Policy check before execution (BACKGROUND capability)
- Audit every invocation (source="background", request_id, outcome)
- Bounded per-task timeout enforced by systemd (TimeoutStartSec=120)

### 2.4 Task Implementations (gate/heartbeat/tasks.py)

Each task is a pure function with bounded execution:

- **health_check**: Call gate.health() + AuditLog.verify()
- **cleanup_runtime**: Call RuntimeIsolation.cleanup_expired()
- **quota_report**: Read QuotaTracker status, write to /secondary/logs/
- **audit_verify**: Call AuditLog.verify(), report chain status

### 2.5 Policy Rules

New rules via build_heartbeat_rules():

- Rule("admin", "BACKGROUND", "background.health_check", allow=True)
- Rule("admin", "BACKGROUND", "background.cleanup_runtime", allow=True)
- Rule("admin", "BACKGROUND", "background.quota_report", allow=True)
- Rule("admin", "BACKGROUND", "background.audit_verify", allow=True)

User role gets NO background rules -> denied.

### 2.6 Systemd Units

**uta-heartbeat.timer** (installed to /etc/systemd/system/):
- AfterBootSec=60s, OnUnitActiveSec=5min
- RandomizedDelaySec=30s (prevents thundering herd)
- WantedBy=timers.target (survives reboot)

**uta-heartbeat.service** (Type=oneshot):
- ExecStart=/srv/dev/UTA/gate/.venv/bin/python -m gate.heartbeat --task health_check
- TimeoutStartSec=120
- User=root (needs access to /secondary/runtime/)

### 2.7 CLI Integration

New module: gate/heartbeat/__init__.py + gate/heartbeat/cli.py

- python -m gate.heartbeat --task health_check
- python -m gate.heartbeat --task cleanup_runtime
- python -m gate.heartbeat --task quota_report
- python -m gate.heartbeat --task audit_verify

### 2.8 Config Additions

New fields in Config (config.py):
- heartbeat_enabled: bool = True
- heartbeat_interval_seconds: int = 300
- heartbeat_tasks: list[str] = [health_check, cleanup_runtime, quota_report, audit_verify]
- heartbeat_max_retries: int = 3

### 2.9 Ingress Schema Update

Add "background" to _SOURCES set in ingress/schemas.py.

## 3. Files to Create/Modify

### New Files
| File | Purpose |
|------|---------|
| gate/heartbeat/__init__.py | Package init |
| gate/heartbeat/runner.py | HeartbeatRunner class (retry, audit, policy) |
| gate/heartbeat/tasks.py | Task implementations (health, cleanup, quota, audit verify) |
| gate/heartbeat/cli.py | CLI entry: --task argument |
| tests/test_heartbeat.py | 35+ tests |
| systemd/uta-heartbeat.timer | systemd timer unit |
| systemd/uta-heartbeat.service | systemd oneshot service |

### Modified Files
| File | Change |
|------|--------|
| gate/config.py | +heartbeat fields |
| gate/policy/engine.py | +build_heartbeat_rules() |
| gate/ingress/schemas.py | +background to _SOURCES |
| gate/__main__.py | +heartbeat CLI routing |
| gate/factory.py | +heartbeat wiring |

## 4. Test Coverage Plan (~35 tests)

| Category | Tests |
|----------|-------|
| HeartbeatRunner (retry, audit, policy) | 10 |
| health_check task | 4 |
| cleanup_runtime task | 4 |
| quota_report task | 4 |
| audit_verify task | 4 |
| Policy rules (background allowed/denied) | 4 |
| CLI integration | 3 |
| Config defaults | 2 |
| **Total F6 new** | **~35** |

## 5. Security Properties

1. **Privilege = interactive, never more**: Background tasks use same Gate pipeline, same policy, no elevated capabilities
2. **BACKGROUND deny-by-default**: Only explicitly listed tasks allowed, user role gets nothing
3. **Audit trail**: Every background invocation recorded with source="background", request_id, outcome
4. **Bounded execution**: Each task has timeout, systemd enforces TimeoutStartSec=120
5. **Retry bounded**: Max 3 retries with exponential backoff, no infinite loops
6. **No AI polling**: Scheduler is systemd timer, tasks are deterministic functions
7. **Clean failure**: Exit 1 on failure, systemd logs to journal, no orphaned processes
8. **Survives reboot**: systemd enable + WantedBy=timers.target

## 6. What F6 Does NOT Do

- Does NOT start a persistent daemon (oneshot + timer only)
- Does NOT add cloud execution to background (cloud remains interactive)
- Does NOT modify ai-gateway.service or llama-server.service
- Does NOT touch /secondary/cache/
- Does NOT grant user role any background capabilities
- Does NOT add AI/LLM calls to scheduler

## 7. Deployment Steps (after approval)

1. Create heartbeat module (runner, tasks, CLI)
2. Add policy rules
3. Add config fields
4. Write tests -> all pass
5. Create systemd units
6. Commit on vox-space
7. Install systemd units: systemctl daemon-reload && systemctl enable --now uta-heartbeat.timer
8. Verify: systemctl status uta-heartbeat.timer
9. Push to GitHub
10. Deploy docs to portal

## 8. AWAITING APPROVAL

This is a design-only preflight report. No code has been written, no services started, no files modified.

Approve to proceed with implementation.
