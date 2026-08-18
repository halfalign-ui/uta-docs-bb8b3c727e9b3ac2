# Proof-of-Run: Pola Tiket di vox-space (2026-08-16)

**Kesimpulan singkat:** OpenCode **native** mendukung alur resepsionis + tiket,
tanpa perlu OpenClaw/Hermes sebagai orkestrator untuk kasus dasar. Semua
diverifikasi live di vox-space dengan `opencode serve` (headless).

## Environment

- Host: `vox-space` (vox, Ubuntu 26.04)
- opencode 1.18.18 di `/usr/local/bin/opencode`
- Provider aktif: **local** (`qwen2.5-7b`) + OpenRouter + Google (auth.json)
- Running via tmux session `oc`, port 3456, hostname 127.0.0.1
- API: `POST /session` buat sesi; `POST /session/{id}/message` body
  `{"parts":[{"type":"text","text":"..."}]}` (bukan `message` key).

## Alur yang diuji (persis visi user)

1. **User kirim tugas** ke resepsionis (sesi `ses_ff7ab5aa...`):
   "Use the Task tool with background=true to spawn a subagent that sleeps 8
   seconds then reports DONE. Just spawn it, do NOT wait."
2. **Resepsionis balas instan dengan tiket:**
   `Background task launched, session id: ses_ff7aab20fffe4Mnkadqjnwv8A0. You
   will be notified when it finishes.`
3. **Resepsionis tetap melayani selama build:** kirim "Quick check: are you
   free?" → jawab "Free to help."
4. **Tiket diakses terpisah (pantau):** POST ke sesi tiket `ses_ff7aab20...`
   "Report your status." → subagent jawab "DONE".
5. **Auto-notify:** setelah 8 detik, resepsionis menerima message sistem
   `<task id="ses_ff7aab20..." state="completed"><summary>Background task
   completed...` → resepsionis membalas "Task completed."

## Detail teknis

- Env var `OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true` di 1.18.18
  **tidak** menyebabkan crash (uji ulang dengan start bersih sukses; kejadian
  "server exited unexpectedly" pertama = race/state lama, bukan flag).
- Body API yang benar untuk message di 1.18.18: `{"parts":[{...}]}` —
  `{"message":{...}}` dan `{"message":{"role","parts"}}` → `BadRequest`.
- Model `qwen2.5-7b` (local) dipakai otomatis; token read cache diulang (7480).

## Relevansi ke arsitektur

- **OpenCode bisa jadi resepsionis + worker sekaligus** (background subagent
  = sesi terpisah + tiket + auto-notify). Ini memangkas kebutuhan orkestrator
  eksternal untuk pola dasar.
- OpenClaw/Hermes tetap relevan sebagai **layer channel** (web/Telegram/WhatsApp)
  atau untuk orkestrasi lintas-proses; lihat `notes/orkestrasi-multi-agent.md`.
- Belum diuji: multi-kali spawn bersamaan, `attach` TUI dari klien ke sesi
  tiket, resume `task_id`, dan batasan token saat banyak worker.

## Setup (untuk reproduksi)

```bash
# di vox-space
export HOME=/home/vox
tmux new-session -d -s oc "opencode serve --port 3456 --hostname 127.0.0.1"
# buat sesi resepsionis
curl -s -X POST http://127.0.0.1:3456/session -H 'content-type: application/json' -d '{"dir":"/tmp/octest"}'
# kirim tugas
curl -s -X POST http://127.0.0.1:3456/session/<R>/message \
  -H 'content-type: application/json' \
  -d '{"parts":[{"type":"text","text":"Use Task background=true ..."}]}'
```

Sesi aktif saat ditulis: `R=ses_ff7ab5aabffekvpup79DrQTG2U`,
`T=ses_ff7aab20fffe4Mnkadqjnwv8A0` (masih bisa dibuka via API sampai tmux
di-kill).
