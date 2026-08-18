# Basis 1 CLI Server Headless — OpenClaw vs OpenCode vs Hermes

> Disusun: 2026-08-16. Tujuan (final): **satu CLI server headless sebagai AI orchestrator** — TANPA TUI/GUI, murni server, input lewat **web** dan **bot Telegram**. Semua klaim diverifikasi dari kode (referensi di map tiap proyek).

## Definisi use-case

- Satu proses server yang bisa dijalankan sebagai service/daemon (bukan app desktop).
- Zero TUI/GUI runtime (boleh ada web dashboard, asal bukan syarat interaksi).
- Input utama: web (dashboard/API) + bot Telegram.
- Orchestrator: memanggil/mengatur model & tool, bukan sekadar chatbot.
- Bonus: WhatsApp (dipakai vox), memory, skills, cron.

---

## OpenClaw — "Multi-channel AI gateway"

Cara jalan sebagai server: CLI `openclaw` (bin `openclaw.mjs`) + channel extension; deskripsi package.json = *"Multi-channel AI gateway with extensible messaging integrations"*. Ada `src/gateway/` (HTTP RPC: `admin-http-rpc`, `board-http`, `control-ui`), extension `webhooks`. Web UI (`ui/`) ada tapi masuk buangan.

| | |
|---|---|
| **Telegram** | ✅ `extensions/telegram/` (channel native, grammy) |
| **Web input** | ⚠️ via extension `webhooks` + HTTP RPC gateway (bukan dashboard bawaan yang matang) |
| **Headless** | ✅ bisa jalan background sebagai gateway channel; tapi web UI/canvas `ui/` harus dibuang |

**Keunggulan**
1. Gateway multi-channel **native** — Telegram, WhatsApp, Discord, Slack, webhooks semua extension; model "satu CLI ngatur banyak channel" memang arsitektur aslinya.
2. `tools.profile`/`tools.deny`/`tools.byProvider` untuk kontrol tool via config (lihat `trim-plan.md`).
3. Tool `systemPromptReport` (`openclaw agent --json`) untuk A/B debloat.
4. WhatsApp extension tipis & bisa dipakai langsung.

**Kekurangan**
1. **Paling boros token**: ±25k token/turn (system prompt 45.8k + schemas 51.6k); system prompt dirakit ulang 25 section tiap turn (`system-prompt.ts` 1.616 baris).
2. Core raksasa & campur aduk: `src/agents/` 1.660 file / 39MB; debloat butuh kerja ekstensif di fork.
3. Build berat: playwright, quickjs-wasi, tree-sitter, clawpdf, node-pty, cua-driver — untuk server headless itu sampah yang harus dipangkas.
4. Web input (dashboard) bukan fokus; lebih ke kanal messaging.
5. Storage/memory tidak sekuat Hermes (memori plugin, bukan SQLite FTS5 terpadu).

---

## OpenCode — otak paling hemat, tapi tanpa channel

Cara jalan sebagai server: **`opencode serve`** = *"starts a headless opencode server"* (`packages/opencode/src/cli/cmd/serve.ts`) — OpenAI-compatible API, auth via `OPENCODE_SERVER_PASSWORD`; `opencode web` = web dashboard; `opencode run` = non-interaktif. Server: `packages/opencode/src/server/server.ts` + routes. UI dibuang.

| | |
|---|---|
| **Telegram** | ❌ **TIDAK ADA** — tidak ada channel messaging sama sekali (grep seluruh `src` kosong) |
| **Web input** | ✅ `opencode serve` (API) + `opencode web` (dashboard React) |
| **Headless** | ✅✅ dirancang untuk ini (`serve` headless, zero TUI) |

**Keunggulan**
1. **Otak paling hemat token**: prompt statis per-provider (`prompt/*.txt` 14 file), section dinamis dibuang kalau kosong, structured output via tool khusus + `toolChoice: required`.
2. Arsitektur bersih & kompak: inti `packages/opencode/src` 3.9MB / ~75k baris; agent inti 480 baris.
3. Storage **SQLite durable** mandiri (`effect-sqlite` + Drizzle, WAL, migrations) — reusable sebagai basis.
4. Server OpenAI-compatible siap (`serve`) — mudah diakses dari web/orchestrator lain.
5. Kandidat buangan jelas (TUI/web/desktop/server/enterprise/control-plane/slack/stats/app).

**Kekurangan**
1. **Tidak ada Telegram/WhatsApp** — harus bikin adapter sendiri, atau pinjam channel dari Hermes/OpenClaw.
2. Fokus = coding agent; memory/skills/cron tidak selengkap Hermes (skill system dasar).
3. Model catalog di-fetch remote (`models.opencode.ai`, TTL 60 mnt) — untuk server headless perlu strategi offline/cache.
4. Bahasa TS + Effect cukup curam untuk modifikasi dalam.
5. Feature V2 runner / native runtime belum terbukti stabil penuh untuk headless (belum terverifikasi).

---

## Hermes — paling lengkap out-of-the-box untuk use-case ini

Cara jalan sebagai server: `hermes` (launcher) → `hermes serve`/`hermes dashboard` (web dashboard + API), `hermes chat -q` (non-interaktif), `hermes gateway` (systemd, messaging). `api_server` = OpenAI-compatible (`gateway/platforms/api_server.py`, 7.6k baris). Web dashboard React (`web/`). UI TUI (`ui-tui/`) & apps GUI bisa dibuang.

| | |
|---|---|
| **Telegram** | ✅ plugin `plugins/platforms/telegram/` (plugin.yaml + adapter.py, first-class) |
| **Web input** | ✅ web dashboard React + `api_server` (OpenAI-compatible) |
| **Headless** | ✅ CLI headless (`-q`), `serve`, gateway systemd; TUI/web bukan syarat |

**Keunggulan**
1. **Cocok total dengan syarat**: Telegram first-class + web dashboard + API server dalam satu CLI server.
2. Memory paling matang: SQLite + FTS5 (`state.db`), MEMORY.md/USER.md (batas 2.2k/1.375 chars), 8 memory-provider plugin, session lineage.
3. Skills system terbaik: 82 bawaan + 115 opsional, SKILL.md portable, auto-create + curator.
4. Toolset per-platform (`hermes-whatsapp` = core saja) + prompt tiered byte-stabil → desain hemat (belum diukur).
5. WhatsApp 2 jalur (Baileys bridge + Cloud API resmi).
6. Python — mudah di-integrasikan ke orchestrator custom vox.
7. Cron, gateway 27 platform, fallback/credential pools, ACP.

**Kekurangan**
1. **Paling besar**: core Python ~670k baris (9.316 file tracked); meski terpisah, debloat tetap butuh seleksi teliti.
2. **Belum diukur token/turn** — klaim hemat belum terbukti (langkah uji pertama).
3. Import chain belum diverifikasi bebas GUI (apakah `import gateway.run` menarik dashboard/curses) → perlu proof-of-run.
4. Baileys WhatsApp unofficial (ban risk) — Telegram aman, WhatsApp perlu Cloud API untuk produksi serius.
5. Packaging Python (uv) + banyak dependency; versi cepat berubah (release mingguan).
6. System prompt & konfigurasi punya banyak tier/opsi — kurva belajar curam.

---

## Matriks untuk use-case ini (1 server headless, web + Telegram)

| Kriteria | OpenClaw | OpenCode | Hermes |
|---|---|---|---|
| Headless server | ⚠️ (channel gateway) | ✅✅ (`serve`) | ✅✅ (`serve`/gateway) |
| Telegram bot | ✅ (extension) | ❌ | ✅ (plugin, first-class) |
| Web input | ⚠️ (webhooks/RPC) | ✅ (serve API + web) | ✅ (dashboard + API) |
| Hemat token | ❌ (25k/turn) | ✅ | ⏳ belum diukur |
| Memory/persistence | ⚠️ | ✅ | ✅✅ |
| Skills | ⚠️ | ⚠️ | ✅✅ |
| Cron | ⚠️ | ⚠️ | ✅ |
| WhatsApp | ✅ | ❌ | ✅✅ |
| Mudah di-trim utk server | ❌ | ✅ | ⚠️ (besar tapi terpisah) |
| Cocok jadi orchestrator | ⚠️ | ✅ (API) | ✅✅ (Python + gateway) |

## Rekomendasi arah (riset — belum final)

1. **Basis utama = Hermes** — paling cocok untuk "1 CLI server, web + Telegram": langsung punya Telegram + web dashboard + API server + memory/skills/cron. Kerja debloat: buang `apps/ web/ ui-tui/ tui_gateway/ website/ contributors/ acp_adapter/` + platform/provider/memory plugin non-inti + seleksi tools (sudah dipetakan di `hermes/notes/map.md`).
2. **Opsi "otak" = OpenCode** — kalau hasil ukur token Hermes mengecewakan, pelajari jalur provider `opencode-zen`/`opencode-go` di Hermes (`plugins/model-providers/`) → Hermes sebagai host, opencode sebagai backend LLM. Atau pakai `opencode serve` sebagai API server terpisah.
3. **OpenClaw = referensi channel/WhatsApp** saja, bukan basis — paling boros & berat untuk server headless murni.

## Langkah uji berikut (butuh data)
1. Ukur per-turn token Hermes (system prompt + toolset `hermes-telegram`/core) — pola `measurements/system-prompt-2026-08-11.md`.
2. Proof-of-run headless di vox: `hermes chat -q`, `hermes serve`, `hermes gateway` (telegram) — cek startup/memori & import chain tanpa GUI.
3. Uji Telegram end-to-end (plugin telegram) + web dashboard.
4. Kalau Hermes oke → buat fork `CustomHermes` + rencana trim (pola `openclaw/notes/trim-plan.md`).
