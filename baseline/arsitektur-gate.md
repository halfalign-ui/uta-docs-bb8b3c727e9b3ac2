# Desain Arsitektur: Orkestrator Berlapis — "Cloud Brain + Local Gate"

> Versi 1 — 2026-08-16. Dokumen fondasi. Dibuat sebelum membongkar
> OpenCode/OpenClaw/Hermes untuk diambil fungsinya.

## 1. Ringkasan

Sistem orkestrasi **3 level**, semua berjalan sebagai agent AI:

- **Level 1 — Resepsionis** (cloud AI): orkestrator **atas**. Selalu online,
  terima input user (web/Telegram), kasih **tiket** ke build agent, tetap
  melayani user, lakukan heartbeat/jadwal.
- **Level 2 — Build Agent** (cloud AI): orkestrator **bawah** (worker). Terima
  tiket dari Level 1, rencanakan pekerjaan, lalu **berikan command/script** ke
  Level 3. Dia tidak mengeksekusi apa pun di device.
- **Level 3 — Local Gate** (local AI): **eksekutor + guardrail**. Punya akses
  device (file, terminal, network, browser), eksekusi command dari
  Level 2, dan **memfilter** apa yang boleh masuk/keluar.

Prinsip kunci: **cloud AI TIDAK PERNAH menyentuh device langsung.** Semua akses
fisik lewat Local Gate. Inilah yang membuat arsitektur aman walau otak
penalaran di cloud.

"Orkestrator di atas orkestrator": Level 1 mengorkestrasi Level 2, Level 2
mengorkestrasi Level 3. Tiap level punya otak cloud sendiri; hanya level
eksekusi paling bawah yang punya sentuhan hardware.

## 2. Diagram hierarki

```
User (web / Telegram)
  │  kirim tugas / tanya
  ▼
┌─────────────────────────────────────────────────────────────┐
│ LEVEL 1 · RESEPSIONIS (cloud AI)                            │
│  - selalu online, terima input multi-channel                │
│  - kasih tiket → spawn build agent (background subagent)    │
│  - tetap melayani user sementara worker jalan               │
│  - heartbeat/jadwal (bangun sendiri kalau perlu)            │
│  - laporkan status build ke user                            │
└───────────────────────────────┬─────────────────────────────┘
        tiket (deskripsi tugas, ID sesi)
                                ▼
┌─────────────────────────────────────────────────────────────┐
│ LEVEL 2 · BUILD AGENT (cloud AI, worker)                    │
│  - terima tiket → buka sesi sendiri (isolated)              │
│  - rencanakan pekerjaan (reasoning)                         │
│  - kirim command/script ke Level 3 (bukan eksekusi sendiri) │
│  - kumpulkan hasil → ringkas → balas ke Level 1             │
└───────────────────────────────┬─────────────────────────────┘
        command/script (dalam kontrak terstruktur)
                                ▼
┌─────────────────────────────────────────────────────────────┐
│ LEVEL 3 · LOCAL GATE (local AI, eksekutor)                  │
│  - akses device: file, terminal, network, browser           │
│  - eksekusi command/script dari Level 2                     │
│  - GUARDRAIL: filter input & output, approval, audit        │
│  - kembalikan hasil (boleh di-redact) ke Level 2            │
└─────────────────────────────────────────────────────────────┘
```

Aliran data: user → L1 (brain cloud) → L2 (brain cloud) → L3 (eksekusi lokal).
Umpan balik: L3 → L2 (hasil) → L1 (ringkasan + notif) → user.

## 3. Peran per level

### Level 1 — Resepsionis (cloud AI)
- **Terima input**: web + Telegram (dari layer channel sendiri; lihat
  `notes/basis-headless-server.md`).
- **Beri tiket**: buat sesi build agent terpisah, kirim deskripsi tugas, beri
  user ID sesi (ticket) supaya bisa dipantau.
- **Tetap melayani**: tidak terblokir saat build jalan — pakai mekanisme
  background/async worker (lihat §5, terbukti di `proof-of-run-vox-space.md`).
- **Heartbeat/jadwal**: bangun sendiri secara periodik (opsional; inilah
  kontinuitas gaya OpenClaw yang diinginkan user) untuk nyapu pending job,
  cek status worker, atau sapa user.
- **Kompetensi model**: wajib model cloud berkualitas (Claude/Gemini) karena
  ini wajah interaksi user.

### Level 2 — Build Agent (cloud AI)
- **Sesi terisolasi**: dapat sesi sendiri (parent = resepsionis), history
  bersih, task_id sendiri. Bisa parallel (banyak worker sekaligus).
- **Reasoning**: semua perencanaan & pemecahan masalah di sini (cloud, kualitas
  tinggi).
- **Keluarkan command/script**: terjemahkan rencana jadi perintah terstruktur
  untuk Local Gate. Bukan langsung `exec`/tulis file.
- **Sintesis hasil**: terima output gate → ringkas → lapor ke resepsionis.
- **Bisa pakai tools cloud**: web search, riset, dsb. — yang tidak menyentuh
  device user.

### Level 3 — Local Gate (local AI, eksekutor)
- **Satu-satunya yang punya akses device**: file system, terminal, network,
  browser.
- **Eksekusi command**: dari Level 2. Model cukup **3B** (bukan 7B) karena
  tugasnya eksekusi + patuh kontrak, bukan reasoning (validasi user: kecil tapi
  aman; guardrail ada di lapisan filter, bukan di kecerdasan model).
- **Guardrail wajib**:
  1. **Allowlist tools/commands** — hanya command yang terdaftar boleh jalan.
  2. **Filter output egress** — apa yang dikirim balik ke cloud: hapus secret,
     token, path sensitif; batasi volume.
  3. **Filter input ingress** — command dari cloud divalidasi dulu (bukan
     raw exec).
  4. **Approval** — operasi berbahaya (mis. hapus, kirim ke luar) butuh
     konfirmasi.
  5. **Audit log** — semua eksekusi tercatat (siapa/apa/kapan).
  6. **Resource limit** — timeout, max output, no fork bomb.

## 4. Protokol antar-level (kontrak)

Tiket, command, dan hasil harus punya format terstruktur (JSON) supaya tiap
level bisa difilter & diaudit.

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

## 5. Mekanisme yang sudah terbukti vs belum

### Sudah terbukti (proof-of-run di vox-space, 2026-08-16)
- OpenCode: resepsionis spawn background subagent, balas instan dengan tiket,
  tetap melayani user, auto-notify saat build selesai. (`proof-of-run-vox-space.md`)
- OpenCode: sesi tiket bisa diakses/pantau terpisah via API.
- Model per-agent di opencode (`agent.ts:71` → `model: {providerID, modelID}`),
  artinya Level 1 bisa cloud, Level 3 bisa local dalam satu instalasi.

### Belum terverifikasi / perlu dibongkar
- Heartbeat & kontinuitas ala OpenClaw (apakah worth biaya token, dan bagaimana
  integrasi ke L1 yang berbasis opencode).
- Guardrail/filter egress-egress di Hermes (apakah ada yang bisa dipinjam).
- Protocol command terstruktur L2→L3 (belum ada di ketiga proyek — ini
  kemungkinan jadi custom layer kita).
- ACP (Agent Client Protocol) sebagai standar komunikasi agent ↔ tool runtime.

## 6. Peta ekstraksi fungsi dari 3 proyek

Target: bongkar tiap proyek, **ambil fungsi** yang cocok per level, buang sisanya.

| Level | Fungsi yang dicari | Sumber kandidat |
|---|---|---|
| L1 | Background/async spawn + auto-notify + ticket | OpenCode `Task {background:true}` (`tool/task.ts`), `BackgroundJob` (`background/job.ts`) — ✅ terbukti |
| L1 | Heartbeat / jadwal / bangun sendiri | OpenClaw `heartbeat: {every, activeHours}` (`claws/schema.ts:249`), cron tool |
| L1 | Multi-channel (web/Telegram/WhatsApp) | OpenClaw `extensions/telegram`, `webhooks`; Hermes `plugins/platforms/telegram`, `gateway/api_server.py` |
| L2 | Sesi worker terisolasi + parallel + summary | OpenCode subagent session; OpenClaw `sessions_spawn`/`sessions_yield`; Hermes `delegate_task` (`tools/delegate_tool.py`) |
| L2 | Child guard (no recursive delegate, blocked tools) | Hermes `DELEGATE_BLOCKED_TOOLS` (`delegate_tool.py:64-75`) |
| L3 | Eksekusi command + process registry | Hermes `process_registry.py` (spawn/poll/wait/kill); OpenClaw `bash-*` tools |
| L3 | Guardrail/approval/filter | Perlu desain custom (belum ada yang lengkap di 3 proyek) |

Catatan: **tidak ada satu proyek pun yang punya semua**. Desain ini sengaja
hybrid: ambil per-level yang terbaik, bangun sendiri yang tidak ada.

## 7. Keputusan desain & alasannya

1. **Cloud = brain, local = gate** — kualitas reasoning cloud untuk kerja
   kompleks; akses device tetap di tangan lokal untuk keamanan & privasi.
2. **Gate pakai model kecil (3B)** — cukup untuk eksekusi kontrak terstruktur;
   hemat VRAM (RTX 4060 8GB) dan hemat biaya. Guardrail di lapisan filter,
   bukan di kecerdasan.
3. **Worker = sesi terisolasi di cloud** — supaya resepsionis nggak kebawa
   context build; bisa parallel; bisa dipantau user via tiket.
4. **Protokol terstruktur (JSON)** — semua level bicara lewat format yang bisa
   diaudit & difilter. Ini kunci agar guardrail L3 bisa bekerja.
5. **Hybrid dari 3 proyek + custom layer** — jangan fork satu proyek utuh;
   petakan fungsi per level, ambil yang dibutuhkan.

## 8. Open questions (belum diputuskan)

- Apakah Level 1 & Level 2 bisa satu instalasi opencode (resepsionis = main
  session, build agent = background subagent)? — kemungkinan YA, sudah terbukti
  dasarnya.
- Di mana heartbeat berjalan: di L1 (cloud, mahal) atau L2 (worker, sesekali)?
  atau pakai mekanisme non-token (cron sistem) yang memicu L1?
- Protokol L2→L3: custom JSON via HTTP, atau pakai **ACP** (standar
  agent→runtime)? — perlu riset.
- Untuk Telegram: pakai OpenClaw extension apa bikin layer sendiri di vox?
- Guardrail L3: deterministic (policy engine) atau AI-assisted (local AI
  menilai)? — kombinasi paling masuk akal.

## 9. Langkah berikut (breakdown)

1. **Bongkar OpenCode** → petakan fungsi L1+L2: background subagent, sesi,
   per-agent model, serve/attach, subagent-permissions. (sebagian sudah)
2. **Bongkar OpenClaw** → petakan fungsi L1: heartbeat, cron, multi-channel,
   sessions_spawn/yield, delivery registry, memory lintas percakapan.
3. **Bongkar Hermes** → petakan fungsi L2+L3: delegate_task, process_registry,
   guardrail/blocked tools, approval flow, api_server.
4. **Riset ACP** → apakah jadi protokol L2→L3.
5. **Desain guardrail L3** → policy engine + approval + audit (custom).
6. **Proof-of-concept** → L1(cloud)+L2(cloud)+L3(local) end-to-end di vox.
