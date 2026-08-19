# Perbandingan 3 Arah — OpenClaw vs OpenCode vs Hermes

> Disusun: 2026-08-16. Sumber: `openclaw/notes/map.md`, `opencode/notes/map.md`, `hermes/notes/map.md` (semua terverifikasi dari kode). Target use-case: **agent headless + WhatsApp** untuk vox.

## Tabel ringkas

| Aspek | OpenClaw | OpenCode | Hermes |
|---|---|---|---|
| Repo | `openclaw/openclaw` (TS, monorepo) | `anomalyco/opencode` (TS, Effect, pnpm) | `NousResearch/hermes-agent` (Python) |
| Lisensi | MIT | MIT | MIT |
| Ukuran inti | `src/` 173MB / 14.179 file (agent 1.660 file) | `packages/opencode/src` 3.9MB / 355 file / ~75k baris | core Python ~670k baris (9.316 file tracked) |
| Bahasa | TypeScript | TypeScript (Vercel AI SDK + native runtime) | Python (+ TS frontend) |
| Agent loop | `src/agents/embedded-agent-runner/` → auto-reply dispatch | `session/processor.ts` (stream) | `agent/conversation_loop.py:1829` (sync loop + thread pool, 8 worker) |
| System prompt | rakit 25 section dinamis/turn (`system-prompt.ts` 1.616 baris) | statis per-provider (`prompt/*.txt`, 14 file / 1.256 baris) + 4 blok dinamis kecil | tiered stable/context/volatile, byte-stabil utk prompt cache |
| Tool | 48 ter-load (schema 51.6k chars) | registry kecil + structured output | 87 registered / 59 core / 35 toolset |
| WhatsApp | extension `extensions/whatsapp/` (adapter tipis) | tidak ada channel | **2 jalur**: Baileys bridge (plugin, unofficial) + WhatsApp Cloud API (resmi) |
| Storage | memory plugin (lancedb dll) | SQLite durable (`effect-sqlite`, Drizzle, WAL) | SQLite + FTS5 (`state.db`), MEMORY.md/USER.md |
| Skills | `skills/` (52) | `skill/` system | 82 bawaan + 115 opsional + curator auto-archive |
| Per-turn token | ±25k (system prompt 45.8k + schemas 51.6k) | hemat (kecil & terstruktur, belum diukur) | belum diukur (toolset per-platform `hermes-whatsapp` = core saja) |
| Headless | ya (channel + core bisa dipisah) | ya (semua UI buang) | ya (gateway/CLI tanpa TUI/web) |

## Dimensi kunci

### 1. Ukuran & kompleksitas
- **OpenClaw** paling besar & paling "boros": agent core 1.660 file di satu folder, system prompt dirakit ulang tiap turn dari 25 section. Ini sumber masalah token yang sudah diukur (±25k/turn tanpa riwayat).
- **OpenCode** paling kompak: inti agent 480 baris + session loop 8.1k baris + prompt statis 1.3k baris. Arsitektur Effect TS, event stream, storage SQLite durable.
- **Hermes** sebesar OpenClaw (bahkan lebih) dalam baris, tapi arsitekturnya **platform-agnostic core + gateway terpisah** — core Python tidak menarik UI; untuk headless WhatsApp hanya butuh subset (agent/, tools/ subset, gateway/, hermes_cli/ subset, state, cron, skills).

### 2. Token efficiency
- OpenClaw: boros — 25 section dinamis, tool catalog 48, memori + bootstrap (60K budget).
- OpenCode: hemat — prompt statis per-provider, section dinamis dibuang kalau kosong, structured output via tool khusus + `toolChoice: required`, compaction/prune output tool (2k chars).
- Hermes: belum diukur, tapi desain mendukung hemat — prompt tiered byte-stabil (prompt cache sacred), toolset per-platform (`hermes-whatsapp` = core), batas memori 2.2k chars, preflight compression di 50% limit.

### 3. WhatsApp
- OpenClaw: extension `whatsapp/` (baileys) — adapter tipis ke `dispatchInboundMessage`; mudah diganti otak.
- OpenCode: **tidak ada channel messaging** — murni coding agent.
- Hermes: dukungan terbaik — Baileys bridge (jalur vox, unofficial) + WhatsApp Cloud API (resmi); 27 platform enum.

### 4. Storage & memori
- OpenClaw: memori plugin; riwayat per-turn besar.
- OpenCode: SQLite durable + migrations, mandiri di `packages/core` — bagus sebagai basis storage.
- Hermes: SQLite + FTS5, WAL, turn lease antarkproses, session lineage, MEMORY.md/USER.md dengan batas char. Paling kaya.

### 5. Skill
- Hermes paling matang: format SKILL.md portable, auto-create, curator, skills hub. OpenClaw punya `skills/`. OpenCode punya skill system dasar.

## Kandidat basis untuk produk akhir (headless + WhatsApp)

| Kriteria | OpenClaw | OpenCode | Hermes |
|---|---|---|---|
| WhatsApp siap pakai | ✅ (bailey) | ❌ | ✅✅ (2 jalur) |
| Hemat token | ❌ (25k/turn) | ✅ | ⏳ (desain oke, belum diukur) |
| Inti ringkas & mudah di-trim | ❌ | ✅ | ⏳ (besar tapi terpisah) |
| Memory/persistence kuat | ⚠️ | ✅ | ✅ |
| Ekosistem skill | ⚠️ | ⚠️ | ✅ |
| Bahasa tim (vox) | TS | TS | Python |

**Kesimpulan sementara (riset):** OpenCode punya "otak" paling hemat & storage bagus tapi **tanpa channel WhatsApp**. Hermes punya WhatsApp + memory + skills terlengkap tapi belum diukur token & runtime-nya. OpenClaw justru kombinasi terburuk (boros + WhatsApp adapter tipis) kecuali dipakai sebagai referensi fitur channel.

Tiga arah eksplorasi yang masuk akal:
1. **A — OpenClaw trim** (fork `CustomOpenClaw` sudah ada): matikan tool/deny + rampingkan system prompt. Tetap boros relatif.
2. **B — OpenCode sebagai otak + Hermes/OpenClaw sebagai channel**: opencode headless sebagai core, gateway WhatsApp dipinjam. Butuh integrasi (Hermes punya provider `opencode-zen`/`opencode-go` — menarik untuk ditelusuri).
3. **C — Hermes sebagai basis**: trim `apps/web/ui-tui` + platform + provider, ukur token/turn. Potensi paling lengkap out-of-the-box.

## Langkah berikut (butuh data sebelum keputusan)
1. **Ukur per-turn token Hermes** (pola seperti `measurements/system-prompt-2026-08-11.md` OpenClaw): system prompt + tool schemas + overhead per turn kosong, toolset `hermes-whatsapp`.
2. **Proof-of-run Hermes headless** di mesin vox (venv terpisah): `hermes chat -q` + `hermes gateway` WhatsApp; ukur startup/memori, cek import chain tidak menarik GUI.
3. **Uji jalur opencode-zen provider Hermes** → apakah bisa memakai opencode sebagai backend LLM.
4. **Cek `import_agent.py` Hermes** — migrasi memori/session dari OpenClaw.
5. Setelah data → keputusan basis (A/B/C) → fork/trim.

## Referensi
- `openclaw/notes/map.md` (commit `f870d93e`)
- `opencode/notes/map.md` (commit `4643e65a`, branch dev)
- `hermes/notes/map.md` (commit `d5773bfc`)
- `openclaw/notes/trim-plan.md` — rencana trim OpenClaw (arsitektur A)
- `notes/basis-headless-server.md` — analisis fokus untuk 1 CLI server headless (web + Telegram), keunggulan & kekurangan tiap proyek
