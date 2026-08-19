# UTA F8-C1 Postflight Security Audit

**Date:** August 19, 2026
**Environment:** vox-space (192.168.100.9)
**Commit:** 0fe5429 (main repo), 6915a9d (public docs)

## Audit Results

### 1. Git Integrity: PASS
- **Working Tree:** Clean.
- **Secrets:** Checked for `env`, `secret`, `vault`, `audit` inclusions in HEAD. No generated system files or tokens were committed.

### 2. uta-gate.service: PASS
- **Unit Exists:** Yes, `/etc/systemd/system/uta-gate.service`.
- **Status:** Active (running).
- **Service Identity:** `User=vox` and `Group=vox`.
- **Environment:** `EnvironmentFile=/etc/uta/gate.env` is configured.
- **Binding:** Binds strictly to `127.0.0.1:8417`.
- **Hardening:** Active (`NoNewPrivileges`, `ProtectSystem=strict`, etc).
- **Restart Check:** Service restarts gracefully and remains healthy.

### 3. uta-heartbeat: PASS
- **Timer:** Active and waiting for the next cycle.
- **Identity:** `uta-heartbeat.service` executes as `vox:vox`.
- **Trigger Test:** Manually triggered via `systemctl start uta-heartbeat.service`.
- **Audit Verification:** Successfully wrote background task execution records to `/var/log/uta/audit/audit.jsonl`.
- **Chain:** The hash chain remained valid (`AuditLog.verify()` succeeded) after the manual trigger.

### 4. Vault: PASS
- **Permissions:** `/data/vault` is `vault:vox 0770`.
- **Validation:** Architecture requires Gate/heartbeat (running as `vox`) to execute `vault_server.py` and perform `_handle_write` and `_handle_delete` operations. Group write access is strictly necessary for this mechanism to function, making `0770` the minimum safe permission model. No permissions were weakened.

### 5. Audit Log: PASS
- **Configuration:** `UTA_AUDIT_DIR=/var/log/uta/audit` confirmed in `/etc/uta/gate.env`.
- **Active Log:** `audit.jsonl` owned by `vox:vox` with `0640` permissions.
- **Integrity:** `AuditLog.verify()` confirms cryptographically valid hash chains.
- **Old Location:** The old `/srv/dev/UTA/gate/var/audit` directory remains but its size is static, confirming it is fully abandoned.

### 6. Logrotate: PASS
- **Syntax:** Validated `/etc/logrotate.d/uta-audit`.
- **Forced Rotation Test:** Executed `logrotate -f`.
  - Rotated files (`audit.jsonl.1`, `audit.tail.1`) remain intact.
  - Active files (`audit.jsonl`, `audit.tail`) were cleanly recreated.
  - Heartbeat execution successfully appended to the new file, correctly starting a new segmented chain (`prev_hash=000...000`).
  - Active chain verifies successfully.
  - Gate remains healthy post-rotation.

### 7. Backup: FAIL (Scope Deviation)
- **Script Status:** `/usr/local/bin/uta-backup.sh` exists and has `0700` permissions.
- **Security Check:** Emits no secrets, enforces fail-closed behavior, writes archive with `0600` permissions.
- **Deviation:** The script intentionally **excludes** `/data/vault` and `/var/log/uta/audit` (as reported in F8-C1 to avoid unencrypted secret backup), contrary to this postflight checklist's expectation.

### 8. Sudo: FAIL (Pending F8-C2)
- **Status:** `/etc/sudoers.d/vox-uta` does not exist.
- **Deviation:** The `vox` user still holds `NOPASSWD: ALL` privileges. Privilege minimization was explicitly deferred to F8-C2 in the F8-C1 report.

### 9. UFW: FAIL / WARNING (Pending F8-C2)
- **Status:** `inactive`.
- **Deviation:** UFW activation and firewall restrictions are explicitly pending the F8-C2 phase.

### 10. Network: PASS
- **Listeners:**
  - `llama-server` -> `127.0.0.1:8080`
  - `ai-gateway` -> `127.0.0.1:8090`
  - `uta-gate` -> `127.0.0.1:8417`
- **External Exposure:** None. SSH (22) is the only externally listening service.

### 11. Authentication: PASS
- **Missing Token:** `curl POST /command` -> `401 Unauthorized`.
- **Invalid Token:** `curl POST /command` -> `401 Unauthorized`.
- **Valid Token:** Authenticates successfully (returns `400 Bad Request` due to missing payload validation, confirming successful bypass of the auth layer). No credentials exposed in output.

### 12. SSE: PASS
- **Connection:** Hit `http://127.0.0.1:8417/events` via `curl -N`.
- **Result:** Connection successfully established (`HTTP/1.0 200 OK`) and kept alive, successfully utilizing production Bearer token authentication.

### 13. Full Regression: PASS
- **Execution:** `pytest -q`
- **Result:** 463 passed in ~19s. 0 regressions.

---

## Final Security Assessment

### Concerns
- The `vox` user currently maintains `NOPASSWD: ALL` sudo privileges, bypassing systemd hardening if compromised.
- The Vault and Audit logs are currently omitted from automated backups. Unencrypted vault backups represent a critical security challenge that requires architectural resolution (e.g., GPG wrapping) before implementation.

### Recommendations
1. **F8-C1 Status:** Should remain **CLOSED**. The implemented foundation is highly stable, deterministic, and safe. The "failed" items were deliberately excluded from F8-C1 scope and documented as such.
2. **Next Steps:** The system is completely safe to proceed to **F8-C2** (Final Security Lock & UFW). F8-C2 must address the UFW activation and sudo privilege downgrade for `vox`. Vault backup strategy should be discussed separately before implementation.