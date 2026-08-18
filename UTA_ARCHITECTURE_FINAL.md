# UTA — FINAL ARCHITECTURE SPECIFICATION (F1)

> Status: F1 DESIGN — proposal final untuk review. Tidak ada implementasi.
> Referensi: `blueprint-uta-v1.md`, `UTA_PROJECT_STATE.md`, `UTA_ARCHITECTURE_DESIGN.md`.
> Tanggal: 2026-08-16.
> Kunci: setiap hal yang tidak pasti ditandai **UNKNOWN**.

---

## 1. Architecture Overview

UTA = orkestrator headless 3-level **Cloud Brain + Local Gate**. Reasoning berat
di cloud; mesin ini hanya menyediakan eksekusi, policy, dan audit yang aman.

```
  USER (Telegram)
     │  pesan bebas
     ▼
┌──────────────────────────────────────────────────────────────────┐
│ L1  RESEPSIONIS (cloud AI)                                       │
│  · wajah interaksi, selalu online, multi-sesi                     │
│  · kasih tiket → spawn L2                                         │
│  · heartbeat: dipicu systemd timer lokal, bukan AI polling        │
└───────────────────────────────┬──────────────────────────────────┘
           tiket (JSON, kontrak terstruktur)
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│ L2  BUILD AGENT (cloud AI, worker terisolasi)                    │
│  · perencanaan & reasoning (cloud)                               │
│  · produksi command/script terstruktur untuk L3                  │
│  · sintesis hasil → ringkas → balas L1                           │
└───────────────────────────────┬──────────────────────────────────┘
           command (JSON, schema ketat, token sesi)
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│ L3  LOCAL GATE (Lokal, deterministik + model opsional)           │
│  ├─ INGRESS      : auth token + validasi skema kontrak           │
│  ├─ POLICY ENGINE: authorization, approval, deny-default         │
│  ├─ TOOL DISPATCH: MCP client → MCP servers lokal                │
│  ├─ OUTPUT       : egress filter (redact) + audit                │
│  └─ (opsional) ORCHESTRATOR MODEL kecil utk intent/summary       │
└───────────────────────────────┬──────────────────────────────────┘
                                ▼
        MCP servers: filesystem · workspace · system · vault
                                ▼
                 RESOURCES (fs, services, vault)
```

Prinsip yang dijaga: cloud tidak pernah akses device; LLM tidak pernah jadi
security boundary; semua keputusan berbahaya ditentukan policy deterministik.

## 2. Component Diagram

```
            L3 LOCAL GATE (satu service, Python 3.12+)
┌──────────────────────────────────────────────────────────────┐
│ api_server (HTTP + SSE)                                       │
│   ├─ POST /command        → ingress: auth + schema            │
│   ├─ POST /approve        → approval gate                     │
│   └─ GET  /events         → SSE stream (status, output)       │
├──────────────────────────────────────────────────────────────┤
│ policy_engine (deterministik, TANPA LLM)                      │
│   ├─ allowlist tools / deny rules                             │
│   ├─ scope per tiket (paths, hosts, services)                 │
│   ├─ approval gate (inline / deny-default bg)                 │
│   └─ schema validator (JSON Schema)                           │
├──────────────────────────────────────────────────────────────┤
│ command_runner (subprocess + systemd-run scope, timeout,      │
│                 MemoryMax/CPUQuota, anti-fork-bomb)           │
├──────────────────────────────────────────────────────────────┤
│ mcp_client (SDK MCP) → filesystem | workspace | system | vault│
├──────────────────────────────────────────────────────────────┤
│ audit_log (append-only, JSONL → /secondary/logs, hash-chain)  │
├──────────────────────────────────────────────────────────────┤
│ secret_redactor (egress filter: pola secret/path → redact)    │
└──────────────────────────────────────────────────────────────┘
  opsional: orchestrator_model (1.5–3B, intent & summary saja)
```

## 3. Trust Boundaries

```
 Cloud L1 ──► Cloud L2 ──► L3 INGRESS ──► POLICY ──► TOOL/MCP ──► RESOURCE
   UNTRUSTED   UNTRUSTED      AUTH         AUTHZ        SCOPE      DATA
```

| Boundary | Isi | Status |
|---|---|---|
| Cloud (L1, L2) | model cloud, reasoning | **UNTRUSTED** untuk akses resource apa pun |
| L3 ingress | proses gate lokal, token | auth + validation di sini |
| Policy engine | aturan deterministik | trust boundary utama eksekusi |
| MCP server | proses tool per-domain | trust boundary per domain, scope sempit |
| Vault | data pribadi | boundary data pribadi (terisolasi maksimal) |

LLM lokal = **bukan** security boundary (komponen opsional di belakang policy).

## 4. Protocol Decision — HYBRID

| Kandidat | Evaluasi |
|---|---|
| Custom JSON | Sederhana, audit total, kontrol schema; non-standar |
| ACP (Agent Client Protocol) | Standar agent↔client, streaming SSE; ekosistem muda, schema belum stabil |
| MCP | Standar tools, ekosistem besar; bukan protokol untuk command/approval antar-agen |
| Hybrid | Memakai yang paling pas per lapisan |

**Keputusan:**

1. **L2 → L3 ingress: custom JSON over HTTP + SSE.**
   - Skema ketat (JSON Schema), versioned, kontrak tunggal.
   - Setiap command punya `ticket_id`, `cmd_id`, scope, `allow`, timeout, budget.
   - Approval inline (field `approval_required`, endpoint `/approve`).
   - Streaming hasil via SSE (`/events`).
   - Auth: bearer token per sesi cloud.
   - Alasan: kontrol penuh atas audit/approval/schema; ACP terlalu muda untuk
     memenuhi kebutuhan kontrak approval & audit tanpa modifikasi besar.
2. **Gate → tools: MCP.**
   - Gate menjadi MCP **client**; setiap tool = MCP server lokal terpisah.
   - Alasan: isolasi tool, plugin-ability, standar industri; policy tetap di gate.
3. **L1 → L2: custom JSON** (keluarga kontrak yang sama: tiket/result).
   - ACP dijadikan **referensi desain** (lifecycle session, SSE event shape)
     dan interface gate dibuat sebagai adaptor agar bisa migrasi ke ACP
     jika ekosistemnya matang (**UNKNOWN** kapan/migrasi tidak wajib).

## 5. Local Model Decision — TIDAK WAJIB DI FASE AWAL

Distingsi tiga lapisan:

| Lapisan | Fungsi | Butuh LLM? |
|---|---|---|
| Deterministic orchestration | routing, tool selection, schema, policy, approval | **Tidak** (rule + allowlist + schema) |
| Model-assisted orchestration | intent free-text, summarisasi hasil, strukturisasi permintaan | Ya, **opsional & kecil** |
| Cloud reasoning | perencanaan/penulisan kode berat | Ya, di L1/L2 (cloud) |

**Keputusan:** inti L3 = **deterministic penuh**. Model lokal **tidak wajib**
untuk F2–F6. Ditambahkan (F7) **hanya jika** ada kebutuhan konkret yang terbukti:
- intent classification dari teks bebas user (jika rule-based gagal di uji),
- summarisasi hasil panjang sebelum dikirim ke cloud.

Spesifikasi bila dipakai:

| Atribut | Nilai |
|---|---|
| Parameter class | 1.5–3B (Q4) |
| Konteks | 4–8K token |
| Concurrency | 1–2 request (single user) |
| Budget | ~2 GiB VRAM (GPU) ATAU ~4 GiB RAM (CPU-only) |
| Peran | intent/summary; **tidak pernah** memberi keputusan keamanan |

Model spesifik: **UNKNOWN** (tidak diunduh/di-benchmark pada fase ini).

## 6. MCP Design

Gate sebagai MCP client; 4 server lokal, deny-by-default:

| Server | Capability | Scope | Default |
|---|---|---|---|
| `filesystem` | baca/tulis file | workspace yang diizinkan per tiket | read-only |
| `workspace` | manage proyek | `/data/workspaces` | read-only |
| `system` | info + service systemd | info read-only; start/stop approval | read-only |
| `vault` | data pribadi | `/data/vault` via policy | **read-only**, write approval |

Aturan:
- Tiap server: path whitelist, tidak bisa keluar scope (uji deny di F3).
- Tidak ada MCP untuk web/browser (konsisten blueprint §8).
- Otentikasi MCP internal: localhost + perms filesystem + token; tidak expose LAN.

## 7. Vault Design

**Keputusan: kombinasi berlapis** — dedicated user + systemd sandboxing + MCP boundary.

| Opsi | Dipakai? | Alasan |
|---|---|---|
| Dedicated user `vault` + perms 0700 | **Ya (lapis 1)** | boundary fs sederhana & kuat; audit via ownership |
| systemd sandboxing (DynamicUser, NoNewPrivileges, ProtectHome, MemoryMax, CPUQuota, RestrictAddressFamilies) | **Ya (lapis 2)** | isolasi runtime tiap service UTA |
| MCP `vault` server | **Ya (lapis 3)** | satu-satunya interface AI ke /data/vault |
| LXC | Tidak (fase awal) | daemon mati, overhead; bisa dipertimbangkan bila butuh isolasi kernel → **UNKNOWN** |
| Docker | Tidak (untuk vault) | daemon root + image overhead; tidak menambah keamanan vs user+perms+systemd utk data statis |

Parameter:
- Lokasi: **`/data/vault`** (baru; belum ada — proposal). Owner `vault:`, mode 0700.
- Default read/write AI: **read-only**. Write hanya lewat approval yang tercatat.
- Approval: operasi write vault → approval eksplisit user; deny-default background.
- Secret handling: secret di file env mode 0600 milik user service; **tidak pernah**
  masuk log atau keluar ke cloud.
- Audit: semua akses vault (read/write) masuk audit log.
- Cloud visibility: hanya hasil yang sudah lewat egress redactor; raw vault tak
  pernah terlihat cloud.

## 8. Security Model

| Fungsi | Terjadi di | Mekanisme |
|---|---|---|
| Authentication | L3 ingress (+ channel L1) | bearer token sesi (cloud); user allowlist (Telegram) |
| Authorization | Policy engine | allowlist tools + scope tiket + deny rules |
| Validation | L3 ingress | JSON Schema kontrak (type, size, nilai) |
| Approval | Policy engine (approval gate) | inline user; deny-default background |
| Auditing | Gate boundary | audit log append-only (JSONL + hash-chain) |
| Secret redaction | Output pipeline | pola secret/path → redact sebelum egress |

Aturan keras:
- LLM (cloud/lokal) bukan security boundary.
- Background/cron tidak pernah menambah privilege vs interaktif.
- Semua command masuk harus lolos schema + allowlist — tidak ada mode "full trust".
- Deny-by-default di setiap lapisan.

## 9. L1 / L2 / L3 Responsibilities

| Level | Lokasi | Tanggung jawab | Model |
|---|---|---|---|
| L1 Resepsionis | Cloud | terima user, tiket, spawn L2, lapor status, heartbeat trigger | cloud besar |
| L2 Build Agent | Cloud | perencanaan, produksi command terstruktur, sintesis hasil | cloud besar |
| L3 Local Gate | Lokal | ingress, policy, approval, dispatch tool, egress redact, audit; (opsional) intent/summary | deterministik (+1.5–3B opsional) |

## 10. Heartbeat / Background Execution Model

**Keputusan: ya — minimal, dengan privilege setara interaktif (tidak pernah lebih).**

- Pemicu: **systemd timer** di sisi gate (deterministik, murah) → event ke L1 saat
  ada kerjaan; **bukan** AI yang bangun sendiri berkala (menghindari biaya).
- Permitted operations: polling status worker, kirim notifikasi ke user, health
  check, retry sederhana, finalisasi audit.
- Approval: background hanya boleh operasi yang *juga* diizinkan interaktif tanpa
  approval; operasi approval-gated **tetap deny** saat tanpa user.
- Resource limits: timeout per task, antrean maks 3, rate-limit notifikasi
  (max 1/10 s per sesi).
- Failure behavior: retry eksponensial (3x, cap 60 s), lalu mark `failed`, notif
  user; tidak ada loop tanpa batas; watchdog restart hanya service gate.
- Audit: seluruh event background masuk audit (penanda `source=background`).

## 11. Resource Budget

Target mesin: 16 GiB RAM · RTX 4060 8 GiB · 1 TB NVMe (`/data`) · 250 GB SATA (`/secondary`).
Prioritas: multi-task, idle rendah, latency dapat diprediksi, isolasi, reliabilitas.

| Komponen | RAM idle | RAM aktif | CPU | VRAM | Storage |
|---|---|---|---|---|---|
| Local Gate (Python, 1 proses) | <50 MiB | 256–512 MiB | 1 vCPU burst | 0 | 0.2 GiB kode |
| MCP servers (4) | ~100 MiB | 256–512 MiB | 0.2 vCPU | 0 | ~0.1 GiB |
| Vault | ~0 | ~0 | 0 | 0 | data pribadi (sekarang kosong) |
| Monitoring/journal | ~100 MiB | ~200 MiB | 0.1 vCPU | 0 | logs → /secondary/logs |
| Model kecil (opsional F7) | — | 0.5–1 GiB | 1 vCPU | ~2 GiB | 1–2 GiB |
| **Total (tanpa model)** | **~300 MiB** | **~1.2 GiB** | ~2 vCPU burst | 0 | <1 GiB |
| **Total (dengan model)** | — | **~2.3 GiB** | ~3 vCPU | ~2 GiB | ~3 GiB |

- Audit log: rollover mingguan, retensi 90 hari → `/secondary/logs` (216 GiB free).
- Backup: snapshot vault + config → `/secondary/backups` (kosong, siap).
- Workspaces: `/data/workspaces` (678 GiB free).
- Latency target: policy path <50 ms; tool dispatch <200 ms; approval inline <2 s (user).
- Isolasi: tiap service `MemoryMax`/`CPUQuota` via systemd; `NoNewPrivileges`.

## 12. Failure Model

| Kegagalan | Perilaku |
|---|---|
| Cloud (L1/L2) offline | Gate tetap hidup; queue command dgn TTL; job ditandai `deferred`; retry saat cloud kembali |
| Gate crash | systemd restart (watchdog); state job di-persist; audit utuh (append-only) |
| Tool/MCP server gagal | error terstruktur `tool_error`; tidak silent; job `failed` |
| Model lokal (jika ada) gagal | downgrade ke deterministic (tanpa intent/summary) — gate tetap fungsional |
| Egress gagal redact (deteksi secret) | command di-block; audit `egress_denied`; tidak kirim ke cloud |
| Resource overlimit (memori/CPU/disk) | proses di-kill oleh cgroup; error teraudit; tidak memakan host |
| Approval timeout (tanpa user) | deny-default; job `denied` |

## 13. Audit Model

- Format: JSONL append-only per hari, hash-chain antar baris (anti-tamper).
- Isi tiap entri: `ts, source, ticket_id, cmd_id, tool, action, decision,
  approver, result_status, redacted_output_ref, resource_used`.
- Lokasi: `/secondary/logs/uta-audit/` (+ mirror di vault untuk `vault` events).
- Secret: nilai secret **tidak pernah** ditulis (diganti `<REDACTED>`).
- Rotasi: harian, retensi 90 hari; akses hanya root/`vault` + user service UTA.

## 14. Implementation Phases (revisi F2–F8)

Re-evaluasi vs desain F0:

| Fase | Isi | Perubahan & alasan |
|---|---|---|
| **F2** Local Gate core **+ guardrail deterministik** | policy engine, schema, ingress/egress filter, approval, audit, command runner — **tanpa LLM, tanpa MCP** | **MERGE** (F2+F3 lama): filter & approval bagian tak terpisahkan dari policy core; pisah memaksa kerja ulang |
| **F3** MCP tool boundary | gate = MCP client; server `filesystem`, `workspace`, `system` | tetap (dari F4 lama) |
| **F4** Vault & secret handling | dedicated user, `/data/vault`, MCP `vault`, approval write, secret migration | **MAJU** (dari F7 lama): boundary data pribadi harus terkunci **sebelum** koneksi cloud (mitigasi eksfiltrasi sejak awal) |
| **F5** Integrasi cloud L1/L2 + channel | protokol tiket/command/hasil, channel Telegram, PoC end-to-end | tetap (dari F6 lama) |
| **F6** Heartbeat & background | systemd timer scheduler, notifikasi, retry, audit bg | **BARU** (mandiri): approval/audit bg perlu fase uji terpisah |
| **F7** Model kecil (opsional) | intent/summary 1.5–3B, hanya jika kebutuhan terbukti | **TURUN & opsional** (dari F5 lama): keputusan §5 = deterministic-first; bukan prerequisite |
| **F8** Hardening & deploy | UFW, sudo minimal, auth server, backup, monitoring, rotasi log | tetap |

Tidak ada fase yang dihapus tanpa sebab; hasil: 7 fase implementasi (F2–F8).

## 15. Explicitly Unresolved / UNKNOWN

1. Model 1.5–3B spesifik (identitas) — diputuskan saat F7 jika dibutuhkan.
2. Channel L1: bot Telegram **baru** vs **adaptor** dari bot existing — keputusan implementasi F5.
3. Apakah instance `opencode serve` existing (port 3456) menjadi basis L1/L2 — **UNKNOWN**; desain tidak bergantung padanya (gate punya API sendiri).
4. Mekanisme token/API-key final untuk ingress (env file vs vault) — diputuskan F8.
5. Migrasi ACP penuh (jika ekosistem matang) — **UNKNOWN**, interface dibuat adaptor.
6. Need isolasi kernel (LXC/container) untuk vault — **UNKNOWN**, dievaluasi bila ada kebutuhan.
7. UX approval di Telegram (inline button vs balasan manual) — detail F5/F6.

---

```
F1 DESIGN: COMPLETE
```
