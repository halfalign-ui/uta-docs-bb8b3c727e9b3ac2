# Cetak Biru UTA (Uni Trajectory Agent) — v1

> Versi bersih untuk diskusi arsitektur. Tanpa tool otomasi web/personal-use dan tanpa
> detail infrastruktur (host/IP/port/nama layanan). Bahasa Indonesia.

## 1. Ringkasan

UTA adalah **orkestrator AI headless** (tanpa antarmuka grafis) dengan arsitektur
3 tingkat: **Cloud Brain + Local Gate**. Otak penalaran ada di cloud (model AI
berkualitas tinggi), sedangkan akses ke perangkat fisik hanya dipegang oleh satu
gerbang lokal yang juga bertindak sebagai **guardrail** (filter + approval + audit).

Tujuan: membangun asisten yang bisa bekerja di perangkat user secara aman dan
otonom, walau otak penalarannya ada di cloud. Cloud tidak pernah menyentuh
perangkat; semua eksekusi lewat gerbang lokal.

## 2. Prinsip kunci

1. **Cloud AI tidak pernah mengakses perangkat langsung.** Semua akses fisik
   (file, terminal, jaringan) hanya lewat Local Gate.
2. **Orkestrator di atas orkestrator.** Level 1 mengatur Level 2, Level 2 mengatur
   Level 3. Setiap level punya otak AI sendiri; hanya level terbawah yang menyentuh
   perangkat.
3. **Protokol terstruktur (JSON).** Komunikasi antar level menggunakan format yang
   bisa diaudit dan difilter — inilah kunci agar guardrail bisa bekerja.
4. **Hybrid dari proyek open source.** Ambil fungsi terbaik per level dari proyek
   yang sudah ada; bangun sendiri yang belum ada (mis. guardrail dan protokol).
5. **Model kecil di level eksekusi.** Level 3 cukup pakai model lokal kecil karena
   tugasnya eksekusi + kepatuhan kontrak, bukan penalaran kompleks.

## 3. Arsitektur 3 level

```
User (web / Telegram)
  │  kirim tugas / tanya
  ▼
┌─────────────────────────────────────────────────────────────┐
│ LEVEL 1 · RESEPSIONIS (cloud AI)                            │
│  - selalu online, terima input multi-channel                │
│  - kasih tiket → spawn build agent (background)             │
│  - tetap melayani user sementara worker jalan               │
│  - heartbeat/jadwal (bangun sendiri kalau perlu)            │
│  - laporkan status build ke user                            │
└───────────────────────────────┬─────────────────────────────┘
        tiket (deskripsi tugas, ID sesi)
                                ▼
┌─────────────────────────────────────────────────────────────┐
│ LEVEL 2 · BUILD AGENT (cloud AI, worker)                    │
│  - terima tiket → buka sesi sendiri (terisolasi)            │
│  - rencanakan pekerjaan (reasoning)                         │
│  - kirim command/script ke Level 3 (bukan eksekusi sendiri) │
│  - kumpulkan hasil → ringkas → balas ke Level 1             │
└───────────────────────────────┬─────────────────────────────┘
        command/script (kontrak terstruktur)
                                ▼
┌─────────────────────────────────────────────────────────────┐
│ LEVEL 3 · LOCAL GATE (local AI, eksekutor)                  │
│  - akses perangkat: file, terminal, jaringan                │
│  - eksekusi command/script dari Level 2                     │
│  - GUARDRAIL: filter input & output, approval, audit        │
│  - kembalikan hasil (boleh di-redact) ke Level 2            │
└─────────────────────────────────────────────────────────────┘
```

### Peran per level

**Level 1 — Resepsionis (cloud AI)**
- Wajah interaksi: terima input user (web/Telegram), selalu online.
- Beri tiket: buat sesi worker terpisah, kirim deskripsi tugas, beri user ID tiket.
- Tetap melayani: tidak terblokir saat worker jalan (mekanisme background).
- Heartbeat/jadwal: bangun berkala untuk menyapu pending job / cek status worker.
- Model wajib cloud berkualitas tinggi (ini wajah interaksi user).

**Level 2 — Build Agent (cloud AI)**
- Sesi terisolasi: history bersih, bisa paralel (banyak worker).
- Reasoning: semua perencanaan & pemecahan masalah di sini (cloud).
- Keluarkan command/script terstruktur untuk Level 3 — tidak mengeksekusi apa pun.
- Sintesis hasil: ringkas output gate → lapor ke resepsionis.
- Boleh pakai tool cloud (web search, riset) yang tidak menyentuh perangkat user.

**Level 3 — Local Gate (local AI, eksekutor)**
- Satu-satunya yang punya akses perangkat: file system, terminal, jaringan.
- Eksekusi command dari Level 2. Model cukup kecil (kandidat: Qwen 7B lokal;
  desain awal menargetkan 3B) karena tugasnya eksekusi + patuh kontrak.
- Guardrail wajib (lihat §7).
- Tools lokal: eksekusi command, operasi file, dll. Tanpa tool otomasi web (§8).

## 4. Protokol antar-level (kontrak JSON)

Semua pesan antar level berbentuk JSON terstruktur agar bisa difilter & diaudit.

### Tiket (L1 → L2)
```json
{
  "ticket_id": "tkt_ab12",
  "task": "Buat web server Python di /tmp/app",
  "goal": "Apa yang harus tercapai",
  "constraints": ["jangan install global", "port 8080"],
  "session": "ses_<id build agent>",
  "parent": "ses_<id resepsionis>",
  "created_at": "..."
}
```

### Command (L2 → L3)
```json
{
  "cmd_id": "cmd_34",
  "ticket_id": "tkt_ab12",
  "tool": "execute_command",
  "args": {"command": "ls -la /tmp/app"},
  "allow": ["execute_command", "file_write"],
  "max_output_bytes": 20000,
  "timeout_ms": 30000
}
```

### Hasil (L3 → L2)
```json
{
  "cmd_id": "cmd_34",
  "status": "ok",
  "output": "<mungkin di-redact>",
  "truncated": false,
  "audit": "audit_99"
}
```

## 5. Model & stack

- **L1 & L2:** model cloud berkualitas (Claude / Gemini / GPT-class). Reasoning
  kompleks & interaksi user ada di sini.
- **L3:** model lokal kecil yang berjalan di perangkat user (kandidat Qwen 7B,
  target desain 3B). Tugas: eksekusi kontrak + kepatuhan, bukan penalaran.
- **Stack runtime:** basis headless dari proyek open source (serve + API sesi +
  background subagent). Guardrail & protokol antar-level adalah lapisan custom.

## 6. Komponen dari proyek open source

Tidak ada satu proyek yang punya semuanya → desain hybrid, ambil per level.

| Level | Fungsi yang diambil | Sumber (pola) |
|---|---|---|
| L1 | Serve headless + API sesi/event, background subagent + auto-notify, per-agent model | OpenCode |
| L1 | Heartbeat/jadwal + cooldown, cron + delivery, channel sebagai adapter | OpenClaw (konsep) |
| L2 | Delegasi child: role leaf/orchestrator, depth cap, budget config-authoritative, summary budget, kontrol (steer/stop/list) | Hermes (pola) |
| L3 | Guardrail berlapis: hardline block, allowlist, deny rules, approval, cron-deny | Hermes (pola) |
| L3 | Process registry: spawn/poll/wait/kill, isolasi resource | Hermes (pola) |
| L3 | Permission inherit + deny default, output truncation | OpenCode |

Catatan: semua diambil sebagai **pola** (diport/diadaptasi), bukan fork utuh.

## 7. Guardrail L3 (6 wajib)

1. **Allowlist tools/commands** — hanya perintah terdaftar yang boleh jalan.
2. **Filter output egress** — apa yang dikirim balik ke cloud: buang secret/token/
   path sensitif, batasi volume.
3. **Filter input ingress** — command dari cloud divalidasi dulu (bukan raw exec).
4. **Approval** — operasi berbahaya butuh konfirmasi; saat tidak ada user (cron/
   background) defaultnya tolak.
5. **Audit log** — semua eksekusi tercatat (siapa/apa/kapan).
6. **Resource limit** — timeout, batas output, anti fork-bomb, isolasi proses.

## 8. Batas arsitektur (yang TIDAK termasuk)

- **Tidak ada otomasi web/browser (scraping).** Alat semacam itu umumnya melanggar
  syarat layanan situs dan bersifat personal use → dikecualikan dari desain.
- **Tidak ada tool otomasi web pihak ketiga / MCP terkait.** Kecuali nanti ada
  izin eksplisit, tidak dijadikan bagian inti arsitektur.
- Fokus UTA: eksekusi di perangkat user yang memang menjadi tanggung jawabnya
  (file, terminal, jaringan), bukan scraping konten web orang lain.

## 9. Roadmap implementasi

1. Riset protokol L2→L3 (custom JSON vs standar ACP) — keputusan desain.
2. Desain guardrail L3 (policy engine + approval + audit) — lapisan custom.
3. Implementasi L3 Local Gate (eksekusi + filter + guardrail + audit).
4. Implementasi L2 Build Agent (delegasi + summary budget).
5. Integrasi channel L1 (web dan/atau Telegram) + heartbeat/jadwal.
6. Proof-of-concept end-to-end L1+L2+L3 di perangkat lokal.
7. Hardening & deploy permanen (service, password, backup).

## 10. Pertanyaan terbuka untuk diskusi

1. **Model kecil untuk L3**: Qwen 7B (dan model lokal sejenis) pada dasarnya
   mengikuti arsitektur model GPT-class (transformers, chat-tuning). Apakah
   pendekatan "model kecil + guardrail deterministik di lapisan filter" sudah
   tepat, atau sebaiknya eksekusi L3 tidak sepenuhnya bergantung pada model AI?
   Bagaimana cara mengukur kecukupan 3B vs 7B untuk tugas eksekusi kontrak?
2. **Stack L3**: port guardrail (Python → TS) agar satu bahasa dengan basis, atau
   jadikan gate service terpisah yang bisa memakai ulang kode guardrail yang sudah
   matang? Trade-off pemeliharaan vs keseragaman.
3. **Protokol L2→L3**: custom JSON via HTTP vs standar **ACP** (Agent Client
   Protocol)? Mana yang lebih aman, ringan, dan auditable?
4. **Heartbeat L1**: model AI bangun sendiri berkala (mahal) vs cron sistem yang
   memicu API L1 hanya saat ada kerjaan (murah)? Simulasi biaya belum dilakukan.
5. **Keamanan cloud→local**: bagaimana memastikan command dari cloud tidak bisa
   dipakai untuk exfiltrasi? Apakah filter egress + audit cukup, atau perlu
   sandbox/container isolasi penuh di L3?
6. **Multi-channel L1**: web dulu atau Telegram dulu? Channel mana yang menentukan
   bentuk tiket & notifikasi?
