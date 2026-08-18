# F2 — LOCAL GATE CORE + GUARDRAIL (Report)

> Status: IMPLEMENTED (F2). Lokasi kode: `/srv/dev/UTA/gate/` (repo UTA).
> Referensi desain: `UTA_ARCHITECTURE_FINAL.md` §2, §4, §8.
> Tanggal: 2026-08-16.

## Ringkasan

Deterministic L3 Local Gate selesai dibangun sesuai F2 scope. **Tanpa LLM**,
tanpa MCP/vault/Telegram/cloud. Python 3.13 (stdlib only — tanpa framework besar).

Pipeline yang diimplementasikan persis sesuai desain:

```
authenticated → schema valid → command known → policy allows
             → approval satisfied → execute → audit → respond
```

## Yang Dibangun

| Modul | Isi |
|---|---|
| `gate/ingress/` | HTTP + SSE server (loopback), validasi skema ketat |
| `gate/auth/` | abstraksi auth; dev = bearer token (dapat diganti tanpa sentuh policy) |
| `gate/policy/` | 6 capability class + policy engine **deny-by-default** |
| `gate/approval/` | state machine (6 state) + store terikat operasi + TTL |
| `gate/runner/` | allowlist command runner: no shell, argv tetap, env scrubbed, timeout, cap output, path scoped ke workspace |
| `gate/audit/` | audit JSONL append-only + SHA-256 hash-chain + sidecar tail + redact secret |
| `gate/health/` | `/health` (liveness), `/ready` (readiness) |
| `gate/core.py` | orkestrasi pipeline (tanpa keputusan keamanan sendiri) |
| `gate/factory.py`, `gate/cli.py` | wiring + CLI (`python3 -m gate`) |

## Verifikasi

- **70 test** (unit + integrasi HTTP) — `python3 -m unittest discover -s tests -t .` → **OK**.
- Cakupan: schema (valid/invalid), auth (401), policy deny-by-default
  (SECRET/NETWORK/BACKGROUND), approval (state machine, binding, expiry,
  non-reusable, deny), runner (unknown command, path escape via symlink,
  truncation, timeout, no-shell), audit (tamper & truncation terdeteksi,
  secret tidak pernah masuk log), pipeline HTTP (health/ready/command/approve/
  SSE/payload 413/malformed JSON).

## Keputusan Kecil (deviasi dari layout diusulkan)

- Tidak ada `gate/tests/` terpisah di dalam package; test diletakkan di
  `gate/tests/` (di luar package) agar tidak ikut ter-install. Layout lain
  mengikuti proposal F2.
- Approval disimpan in-memory (sesuai F2: belum ada persistence layer);
  `GateServer` memakai `ThreadingHTTPServer` dari stdlib.
- Sidecar `audit.tail` ditambahkan agar **truncation di akhir log** terdeteksi
  (hash-chain saja tidak mendeteksi pemotongan ujung).

## Verifikasi Manual (smoke)

- `python3 -m gate --print-policy` → menampilkan 10 rules (admin + user),
  READ workspace / EXECUTE echo / WRITE remove_file (approval).
- Server + `/health` diuji dalam test integrasi (port ephemeral, loopback).

## Keamanan

- Deny-by-default di semua lapisan; tidak ada mode "full trust".
- `SECRET`, `NETWORK`, `BACKGROUND` selalu tolak (belum ada rule).
- Path escape (termasuk symlink) ditolak di validation (400) sebelum policy.
- Approval tidak pernah berasal dari natural language; terikat `request_id +
  command + capability + arg_hash + requester + TTL`.
- Secret (token auth) di-redact dari audit; tidak pernah ke log.

## Status

```
F2: COMPLETE
SYSTEM CHANGES: NONE (hanya file baru di repo; tidak ada service/produksi diubah)
INSTALLATIONS: NONE (stdlib Python)
PRODUCTION SERVICES: UNCHANGED
```

## Next

F3 — MCP tool boundary (gate sebagai MCP client; server filesystem/workspace/
system). Menunggu approval eksplisit.
