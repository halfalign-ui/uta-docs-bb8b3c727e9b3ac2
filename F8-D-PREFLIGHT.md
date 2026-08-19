# UTA F8-D Preflight Report: Roadmap Reconstruction

## A. Official F8-D Definition
There is no standalone, formal document explicitly defining "F8-D" in the repository. Instead, F8-D is logically inferred as the terminal sub-phase required to close out the "Hardening & Deploy" phase (F8) by addressing the remaining technical debt explicitly deferred in F8-C1 and F8-C2.

## B. Evidence & Supporting Documents
1. **`UTA_ARCHITECTURE_FINAL.md`**: Defines F8 scope as: `UFW, sudo minimal, auth server, backup, monitoring, rotasi log`.
2. **`F8-C1-REPORT.md` / `F8-C1-POSTFLIGHT-AUDIT.md`**: Explicitly document a deviation where Vault (`/data/vault`) and Audit (`/var/log/uta/audit`) directories were excluded from automated backups due to the lack of an encryption-at-rest mechanism.
3. **`F8-C2-REPORT.md`**: Concludes with "GO for F8-D" after completing UFW and Sudo lockdown.

## C. Current Readiness
**READY**. The system baseline is stable at commit `8c3674e`. The Gate service, heartbeat timer, Vault MCP operations, and auth mechanisms are fully functional under the strict OS boundaries imposed in F8-C2. Regression testing yields 463 passing tests with 0 regressions.

## D. Already-Completed Work (Accidental or Previous Phases)
Of the 6 deliverables mandated for F8:
- **Auth Server**: Completed in F8-B (Token Ingress).
- **Rotasi Log**: Completed in F8-C1 (`logrotate`).
- **UFW**: Completed in F8-C2.
- **Sudo Minimal**: Completed in F8-C2.
- **Monitoring (Partially/Accidentally Complete)**: Programmatic health monitoring was satisfied in F8-B via the `/health` and `/ready` endpoints. Background health monitoring was satisfied in F6 via `heartbeat.health_check`.
- **Backup (Partially Complete)**: Non-sensitive configuration backup (`uta-backup.sh`) was implemented in F8-C1.

## E. Remaining Work (Scope of F8-D)
The explicit and only objective of F8-D is **Encrypted Backup Completion**:
1. Implement a secure wrapper (e.g., GPG public-key cryptography or symmetric encryption) to allow safe archiving of highly sensitive data.
2. Update `/usr/local/bin/uta-backup.sh` to include `/data/vault` and `/var/log/uta/audit`.
3. Ensure no secrets (or encryption keys) are leaked in process arguments, `stdout`, or logs.

## F. Risks / Blockers
- **Key Management Bootstrap**: If using symmetric encryption or GPG, the encryption key/passphrase must be supplied to the backup script securely without creating a circular dependency where the backup script's own secrets are exposed. A GPG Public Key approach is recommended, as it requires no secret key on the Gate itself.
- **Privilege Boundaries**: The backup script currently runs as `root` (via `sudo`). Modifying it must respect the strict `vox` sudo allowlist established in F8-C2, meaning `vox` cannot arbitrarily execute or modify the backup script unless explicitly permitted.

## G. Exact Test Plan
1. Generate the necessary encryption configuration (e.g., GPG receiver public key).
2. Execute the updated `/usr/local/bin/uta-backup.sh`.
3. Verify the resulting archive includes `vault` and `audit` directories.
4. Verify the archive cannot be read via standard `tar` (must fail/be encrypted).
5. Attempt a manual decryption drill in a temporary directory to verify data integrity.
6. Verify no interactive password prompts break the automated flow.

## H. Exit Criteria
- `uta-backup.sh` fully backs up configurations, Vault secrets, and Audit logs.
- Archive contents are encrypted at rest.
- Script executes seamlessly and fails closed on error.
- Regression tests (463 tests) continue to pass.

## I. Remaining F8 Documentation Debt
- Finalize the F8 completion status in `docs/UTA-PLAN.md`.
- Ensure all architecture docs reflect the final production state before moving to Project Wrap-up.

## J. GO / NO-GO Recommendation
**GO**. The scope is narrow, well-understood, and strictly operational. F8-D will serve as the final implementation task for the UTA Local Gate.