# UTA F8-D Status: BLOCKED / NO-GO

## 1. BLOCKER — ENCRYPTION KEY INFRASTRUCTURE
- No suitable backup-recipient public key exists on `vox-space`.
- Generating a long-lived key locally is explicitly prohibited by the F8-D implementation constraints, as it forces the invention of an undocumented, unsafe key-storage mechanism.
- Implementation requires an administrator-provided backup recipient public key (e.g., a GPG public key) to be imported into the host's keyring.
- The corresponding private key must remain entirely outside `vox-space` to guarantee that backups remain encrypted at rest even if the host is fully compromised.

## 2. BLOCKER — PRIVILEGE AUTHORITY
- The primary backup script entrypoint, `/usr/local/bin/uta-backup.sh`, is owned by `root:root` with `0700` permissions.
- As a direct result of the successful F8-C2 security hardening, the `vox` service account intentionally possesses no unrestricted `sudo` privileges. 
- The F8-C2 sudo policy must **NOT** be weakened merely to unblock development.
- Root/admin intervention is strictly required to modify or deploy updates to the system-owned backup script.

## 3. CURRENT SECURITY STATE
- **F8-C1**: COMPLETE
- **F8-C2**: COMPLETE
- **Regression**: 463 tests passing (0 regressions)
- **UFW**: Active (deny incoming, allow outgoing, SSH 22 allowed)
- **Sudo**: Minimized (restricted exclusively to specific UTA `systemctl`/`journalctl` commands)
- **Services**: Gate and heartbeat are running under their intended strict service boundaries as `vox:vox`
- **Vault**: Permissions remain unchanged (`vault:vox 0770`)
- **Audit Logging**: Fully active and unchanged

## 4. REQUIRED ADMINISTRATOR INPUT
The only prerequisites for resuming F8-D are:
1. **Provide Key Material**: Provide/import the public key of the intended backup recipient.
2. **Provide Deployment Authority**: Provide a controlled root/admin execution path for modifying/deploying `/usr/local/bin/uta-backup.sh`.

*(Note: Do NOT restore unrestricted sudo to `vox` to achieve this.)*

## 5. RESUME CONDITION
F8-D may resume **only after**:
- The backup recipient public key is available on the system.
- Its fingerprint has been independently verified.
- The administrator confirms where the corresponding private key is safely stored off-server.
- Root/admin deployment authority is available to apply the script updates.
- No private key material has been placed on `vox-space`.

---
**Final Status:** F8-D = BLOCKED / NO-GO
**Reason:** Secure encryption key custody and controlled root deployment authority are external prerequisites, not implementation problems that should be solved by weakening UTA's security model.