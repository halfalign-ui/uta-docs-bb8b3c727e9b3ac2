# UTA F8-D Postflight Audit: Recovery & Encrypted Backup

**Date:** August 19, 2026
**Environment:** vox-space (192.168.100.9)

## Audit Status Overview
**FINAL DECISION:** GO / COMPLETE

---

## 1. Script Deployment: VERIFIED
- `/usr/local/bin/uta-backup.sh` exists on the host.
- Ownership is strictly `root:root` with `0700` permissions.
- Content precisely matches the staged F8-D implementation, enforcing safe backup mechanisms without compromising privilege boundaries.

## 2. GPG Recipient: VERIFIED
- The configured recipient fingerprint inside the script exactly matches the administrator-provided public key imported into the keyring (`2C6E21C59BBDD3F86C7B8E747BE187E3E6BFD87E`).
- No private key operations or generation occurred on `vox-space`.

## 3. Backup Execution: VERIFIED
- A read-only testing execution was directed against a temporary output location (`/tmp/uta-backups`).
- The script successfully completed its execution with a `0` exit code.
- The resulting artifact is correctly permissioned at `0600`.
- The temporary plaintext staging directory (`/tmp/uta_backup_stage_*`) was cleanly wiped post-execution.

## 4. Encryption Verification: VERIFIED
- The output artifact is verifiably a PGP RSA-encrypted session key file.
- Direct extraction attempts (`tar -tf`) immediately fail, confirming no plaintext data leakage exists within the final artifact.
- No decryption was attempted locally.

## 5. Scope Verification: VERIFIED
- Code inspection of the `tar` command invocation confirms explicit inclusion of:
  - `/etc/uta/`
  - `/data/vault/`
  - `/var/log/uta/audit/`
- The script enforces explicit exclusion of `/secondary/runtime/` and ephemeral runtime sockets (`*.sock`).

## 6. Secret Leakage Audit: VERIFIED
- Standard output and standard error logs emit only high-level timestamped metadata (e.g., `[INFO] 2026-08-19T15:39:03+00:00 - Encrypting snapshot with GPG...`).
- Cryptographic shredding (`shred -u`) securely destroys the unencrypted intermediate tarball prior to script exit, guaranteeing zero plaintext persistence on disk.

## 7. Regression Result: VERIFIED
- Command: `.venv/bin/pytest -q`
- Result: **463 tests passed (0 regressions).**
- Execution time: ~18.9s.

## 8. Recovery Documentation: VERIFIED
`docs/F8-D-REPORT.md` correctly outlines the exact security posture required for disaster recovery:
- Administrator private-key offline custody is formally mandated.
- Safe public-key encryption mechanics are documented.
- The offline recovery/decryption procedure is detailed.
- Explicit warnings strictly prohibit the transfer of the private key to `vox-space`.

## Security Notes
The pipeline robustly satisfies all architectural constraints set for the final F8 Hardening & Deploy phase. By relying strictly on asymmetric GPG cryptography for the backup payload, the system avoids generating a circular security problem where backup credentials themselves would need to be securely persisted on the host. F8-D successfully closes the last open security debt item from earlier phases.