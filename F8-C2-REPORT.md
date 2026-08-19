# UTA F8-C2 Report: OS Security Hardening

## Overview
Phase F8-C2 executed the final OS-level security lockdowns for the UTA Gate. This phase enforced the minimal operational privileges for the `vox` service account and activated the system firewall to strictly align with the local-only architectural topology.

## Changes Implemented

### 1. UFW Firewall Activation
- **State Before**: `inactive`.
- **State After**: `active`.
- **Policy**:
  - `Default: deny (incoming), allow (outgoing)`
  - `22/tcp ALLOW IN (Anywhere)`
- **Topology Enforcement**:
  - `uta-gate` (8417), `llama-server` (8080), and `ai-gateway` (8090) remain strictly bound to `127.0.0.1`.
  - The firewall deliberately **omits** external allow rules for these ports, enforcing the local-only constraint at the OS network boundary.

### 2. Sudo Privilege Minimization
- **State Before**: The `vox` user held unrestricted root privileges via `(ALL) NOPASSWD: ALL` defined in `/etc/sudoers.d/vox`.
- **State After**: Unrestricted access revoked.
- **Implementation**: Created `/etc/sudoers.d/vox-uta` containing a strict, absolute-path allowlist.
- **Exact Allowlist**:
  ```
  vox ALL=(root) NOPASSWD: /usr/bin/systemctl status uta-gate.service, \
      /usr/bin/systemctl restart uta-gate.service, \
      /usr/bin/systemctl start uta-gate.service, \
      /usr/bin/systemctl stop uta-gate.service, \
      /usr/bin/systemctl status uta-heartbeat.service, \
      /usr/bin/systemctl restart uta-heartbeat.service, \
      /usr/bin/systemctl start uta-heartbeat.service, \
      /usr/bin/systemctl stop uta-heartbeat.service, \
      /usr/bin/systemctl status uta-heartbeat.timer, \
      /usr/bin/systemctl restart uta-heartbeat.timer, \
      /usr/bin/systemctl start uta-heartbeat.timer, \
      /usr/bin/systemctl stop uta-heartbeat.timer, \
      /usr/bin/journalctl -u uta-gate.service, \
      /usr/bin/journalctl -u uta-heartbeat.service
  ```

## Security Validation

### External Listening Sockets
Verified via `ss -lntup`. The only externally listening socket is SSH (Port 22, `0.0.0.0:*` and `[::]:*`). All UTA application processes correctly bind only to local loopback addresses (`127.0.0.1`).

### Sudo Safety & Lockout Prevention
The transition to restricted `sudo` was validated via a background fallback mechanism that proved the syntax and capabilities of the `vox-uta` ruleset before permanently purging the unrestricted `vox` sudoers file. SSH access remains intact and secure.

### Negative Testing (Sudo)
- **Arbitrary commands**: `sudo -n cat /etc/shadow` -> **REJECTED** (Interactive authentication required / fails).
- **Unauthorized systemctl**: `sudo systemctl is-active uta-gate.service` -> **REJECTED** (Only `status`, `start`, `stop`, `restart` are explicitly allowed).

## System Validation & Regression
- **Service State**: Both `uta-gate.service` and `uta-heartbeat.timer` remain active and healthy.
- **Audit Chain**: Cryptographic continuity remains perfectly valid (`AuditLog.verify() == True`).
- **Network Stack**: SSE streaming and token authentication continue functioning perfectly on the local network interface.
- **Test Suite**: 463 tests passing, 0 regressions.

## Rollback Instructions
- **Firewall**: Execute `sudo ufw disable` (requires a different user with root privileges, as `vox` can no longer execute this).
- **Sudo**: Restore `vox ALL=(ALL) NOPASSWD: ALL` to `/etc/sudoers.d/vox` using a root account.

## Remaining F8 Debt / Exclusions
- The Backup foundation script `/usr/local/bin/uta-backup.sh` explicitly excludes Vault and Audit logs because they contain sensitive material lacking an encryption-at-rest wrapping mechanism. Solving this requires architectural enhancements (e.g., GPG integration) beyond OS hardening.

## Conclusion
**GO for F8-D**. The environment is fully locked down and aligns with all local-only, zero-trust deployment principles. F8-C is definitively closed.