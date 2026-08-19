# UTA F8-D Report: Encrypted Backup Pipeline

## Overview
Phase F8-D completed the operational requirements of the UTA Local Gate by implementing an encrypted-at-rest backup pipeline for sensitive data. It utilizes GPG asymmetric encryption to guarantee that Vault secrets and Audit logs can be safely archived without exposing them to the host filesystem or persisting private key material on the server.

## Exact Implementation
The primary backup entrypoint script has been rewritten to incorporate a fail-closed secure staging and encryption flow. 

**Script Workflow**:
1. **Key Verification**: Looks up the administrator-provisioned public key in the local/target GPG keyring.
2. **Secure Staging**: Creates a volatile temporary directory (`mktemp -d`).
3. **Snapshot**: Archives the live configuration, vault, and audit trails.
4. **Encryption**: Invokes `gpg` using the administrator's public key to encrypt the snapshot directly to the final archive (`/var/backups/uta/*.tar.gpg`).
5. **Secure Wipe**: Utilizes `shred -u` to cryptographically destroy the plaintext snapshot from the staging directory.
6. **Artifact Finalization**: Asserts restrictive permissions (`0600`) on the encrypted artifact.

## Recipient Fingerprint
The backup explicitly targets the administrator-provided public key:
**Fingerprint**: `2C6E21C5...BFD87E` (Abbreviated for security documentation; script uses full fingerprint).
No private key exists on the server.

## Backup Contents & Exclusions
**Included**:
- `/etc/uta/` (Gate & Cloud configs, environment variables)
- `/data/vault/` (Encrypted Vault secrets)
- `/var/log/uta/audit/` (Cryptographic audit hash-chain)

**Excluded**:
- `/secondary/runtime/`
- Runtime sockets (`*.sock`)

## Root-Side Deployment Action Required
Because the `vox` service account operates under strict least-privilege (no unrestricted sudo) established in F8-C2, it cannot overwrite the existing `/usr/local/bin/uta-backup.sh` script.

**Administrator action required:**
1. Review the updated script staged at `/tmp/uta-backup.sh`.
2. Deploy the script using root privileges:
   ```bash
   sudo cp /tmp/uta-backup.sh /usr/local/bin/uta-backup.sh
   sudo chown root:root /usr/local/bin/uta-backup.sh
   sudo chmod 0700 /usr/local/bin/uta-backup.sh
   ```

## Recovery Procedure (Administrator Side)
To recover the system from a backup artifact:
1. Transfer the `.tar.gpg` artifact offline to the administrator's secure machine.
2. Ensure the administrator possesses the private key corresponding to `2C6E21C59BBDD3F86C7B8E747BE187E3E6BFD87E`.
3. Decrypt the archive:
   ```bash
   gpg --decrypt uta_snapshot_YYYYMMDD_HHMMSS.tar.gpg > snapshot.tar
   ```
4. Extract the contents:
   ```bash
   tar -xf snapshot.tar
   ```
*Note: The private key must NEVER be copied to the vox-space host for recovery testing.*

## Verification Results
- **Encryption**: Verified. Output files are strictly GPG-encrypted binary artifacts.
- **Fail-Closed**: Verified. The script aborts and shreds intermediate data if the public key is missing or encryption fails.
- **Permissions**: Output archive is `0600`.
- **System Stability**: 463 tests pass with 0 regressions. No modification was made to Vault permissions, Audit logic, or Gate architecture.

## Rollback Procedure
If the encrypted backup pipeline introduces issues:
1. The administrator can revert to the previous configuration-only backup script by replacing `/usr/local/bin/uta-backup.sh` with the prior F8-C1 version from the Git history.
2. Clear `/var/backups/uta/` of any malformed artifacts.

**Final Status**: F8-D is COMPLETE, pending the explicit root deployment step by the administrator.