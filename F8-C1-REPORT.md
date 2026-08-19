# UTA F8-C1 Report: OS Hardening & Operations

## Overview
Phase F8-C1 focused on OS-level hardening, privilege minimization, and operational foundations for the production Gate. All changes preserve the F7 architecture and strict capability policies. No logic or access controls were fundamentally redesigned; instead, the deployment environment was tightened around the existing security boundaries.

## Changes Implemented

### 1. Audit Log Migration
- **Location**: Migrated from `./var/audit` to `/var/log/uta/audit`.
- **Permissions**: `0750 vox:vox` for the directory, `0640` for logs.
- **Integrity**: The hash chain was cryptographically verified post-migration. `uta-heartbeat.timer` was temporarily paused during migration to ensure no split-brain updates occurred.
- **Config**: `/etc/uta/gate.env` now explicitly sets `UTA_AUDIT_DIR=/var/log/uta/audit`.

### 2. Vault Permissions
- **Permissions**: `/data/vault` changed to `0770 vault:vox`.
- **Justification**: The F4 MCP architecture spawns `vault_server.py` as a subprocess running as `vox`. To support approved `_handle_write` and `_handle_delete` operations for secrets, `vox` requires filesystem write access. `0770` is the minimum safe group permission that accommodates this without broadening access to untrusted users.

### 3. Service Privilege Downgrade (uta-heartbeat)
- **Previous**: Ran as `root:root`.
- **Current**: Runs as `vox:vox`.
- **Integration**: Updated to use `EnvironmentFile=/etc/uta/gate.env` for auth tokens and configuration parity with the Gate.

### 4. Production Service (uta-gate.service)
- **Daemon**: Created `/etc/systemd/system/uta-gate.service` to start the Gate CLI (`python -m gate`).
- **Privileges**: Runs as `vox:vox`.
- **Hardening Applied**:
  - `NoNewPrivileges=true`
  - `ProtectSystem=strict`
  - `ProtectHome=true`
  - `PrivateTmp=true`
  - `ProtectKernelTunables=true`, `ProtectControlGroups=true`, `ProtectKernelModules=true`
  - `RestrictSUIDSGID=true`, `LockPersonality=true`
- **Access**: Uses `ReadWritePaths` for Vault, Audit, Workspace, and secondary Runtime logs.
- **Network**: Binds exclusively to `127.0.0.1:8417` as per local gate requirements.

### 5. Logrotate Foundation
- **Config**: Created `/etc/logrotate.d/uta-audit`.
- **Design Strategy**: Configured for 30 daily rotations. Because the current AuditLog cryptographically links inter-file events, rotation intentionally creates **segmented chains**. Each segment is internally contiguous and verifiable, but continuity resets across segments. Restarts the Gate process post-rotation to gracefully initiate the new segment.

### 6. Backup Foundation
- **Script**: Created `/usr/local/bin/uta-backup.sh`.
- **Scope**: Backs up configuration `/etc/uta` only.
- **Security constraints**: Fails closed, executes with `umask 077` and sets archive permissions to `0600`. Emits no secrets to stdout. Does not back up Vault (which would require a separate GPG-encrypted mechanism).

## Testing & Verification
- **Test Suite**: 463 tests passing, 0 regressions.
- **Gate Startup**: Successfully verified Gate starts and serves requests locally.
- **Heartbeat Startup**: Verified heartbeat task completes successfully without root privileges.

## Remaining F8-C Work (F8-C2)
- UFW rules activation (allow 22, 8417 internally, enable).
- Minimal `sudo` permissions via `/etc/sudoers.d/vox-uta` instead of `NOPASSWD: ALL`.
