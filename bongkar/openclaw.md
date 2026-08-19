# OpenClaw — Bongkar untuk Ekstraksi (Fokus L1: Kontinuitas & Channel)

> 2026-08-16. Upstream `f870d93e`, clone `/srv/dev/upstream/openclaw`.
> Tujuan: ambil fungsi yang dibutuhkan arsitektur 3-level (`notes/arsitektur-gate.md`),
> fokus **Level 1 (resepsionis)**: heartbeat, cron, channel, spawn/yield, memory.
> OpenClaw TIDAK dipakai utuh (408MB source, 48 tool/per-turn ~25k token) — hanya
> diambil pola-pola tertentu. Semua referensi `file:line` = commit `f870d93e`.

## Ringkasan

OpenClaw punya yang OpenCode **tidak punya**: (1) **heartbeat runner** yang bangun
agent sendiri secara periodik, (2) **cron tool** untuk job terjadwal, (3)
**subagent registry** yang tahan banting (persist, restore, delivery), (4)
**multi-channel** sebagai extension adapter. Ini persis "kontinuitas" yang user
inginkan di L1 — tapi di OpenClaw biayanya mahal (per-turn ~25k token). Kita ambil
**pola & config-nya**, bukan kodenya.

## 1. Heartbeat runner — kontinuitas "bangun sendiri" (L1)

File: `src/infra/heartbeat-*.ts` (~50 file, banyak test). Inti:
`heartbeat-runner.ts`, `heartbeat-runner-scheduler.ts`, `heartbeat-schedule.ts`,
`heartbeat-wake.ts`, `heartbeat-runner-run.ts`.

- **Config**: `heartbeat: { every, activeHours: {start, end, timezone} }`
  (`src/claws/schema.ts:249-...`). `every` = durasi (default unit menit),
  `activeHours` = jam aktif (mis. 08:00–24:00). Ini "jadwal bangun".
- **Scheduler**: `startHeartbeatRunner` (`heartbeat-runner-scheduler.ts:64`) —
  tiap agent punya `intervalMs` + `phaseMs` sendiri; wake dilakukan **per-agent
  secara concurrent** (`Promise.all`) supaya satu agent lambat tidak menelan yang
  lain (komentar di `heartbeat-runner-scheduler.ts:349-354`).
- **Wake reasons**: `interval` (terjadwal), manual, `exec-event` (event dari proses
  eksekusi selesai → agent di-bangunkan untuk lanjut). Filter: `heartbeat-events.ts`.
- **Skip logic & cooldown**: `heartbeat-cooldown.ts` (`shouldDeferWake`),
  `heartbeat-wake-policy.ts` — jangan tight-loop; ada `HEARTBEAT_SKIP_NO_PENDING_EVENT`,
  retryable-skip. Ini mencegah biaya token berlebih.
- **Output**: `heartbeat-runner-delivery.ts` + `heartbeat-summary.ts` —
  hasil heartbeat dikirim ke channel (delivery target), bukan selalu balasan penuh.
- **Penting**: heartbeat **bisa disabled** (`areHeartbeatsEnabled/setHeartbeatsEnabled`,
  `heartbeat-wake.ts`) dan ada **cooldown** — jadi tidak selalu "setiap interval
  bayar token penuh". Namun tetap ada biaya LLM per wake.

**Yang diambil (pola)**: interval + active-hours + cooldown + skip-no-pending.
**Yang tidak diambil**: seluruh runner (kita akan tiru ringkas, atau jalankan via
cron sistem yang memicu API L1 — lebih murah dari heartbeat LLM).

## 2. Cron tool — job terjadwal (L1/L2)

File: `src/agents/tools/cron-tool.ts` + `src/cron/*`.

- Tool `cron` mengelola job terjadwal: create/patch, delivery target, wake/run
  actions, reminder payload normalization (`cron-tool.ts:1-8`).
- Ada CLI (`src/cli/cron-cli.ts`) + subsystem `src/cron/` (types, delivery-context,
  delivery-target-validation, normalize, webhook-url).
- **Perlu dibedakan**: cron sebagai **sistem penjadwalan** (bisa ditiru dengan
  cron sistem/`node-cron` di server kita) vs cron sebagai **tool yang dipanggil
  agent** (model menyuruh agent lain jalan di waktu tertentu — pola "orkestrasi").

**Yang diambil (pola)**: konsep job terjadwal + delivery context (hasil dikirim
ke channel/target spesifik). Implementasi bisa pakai cron sistem biasa.

## 3. Subagent spawn/yield + registry (L2 handoff)

File: `src/agents/tools/sessions-spawn-tool.ts`, `sessions-yield-tool.ts`,
`agents-wait-tool.ts`, `agents-list-tool.ts`,
`src/agents/subagents/spawn/subagent-spawn.ts`, `registry/subagent-registry.ts`.

- `spawnSubagentDirect` (`subagent-spawn.ts:97`) — alur lengkap:
  validate request → resolve child plan → buat **initial session anak** →
  prepare context → bind thread/delivery → register run → `launchChildRun`
  (`subagent-spawn.ts:374`) → panggil gateway subagent. Ada `sandbox:
  "require"|"inherit"` (`subagent-spawn.ts:102`).
- `sessions_spawn` params: task/label/agentId/sandbox/thread/context-mode; mode
  swarm untuk parallel fan-out (`sessions-spawn-tool.ts:205`).
- `sessions_yield` = **akhiri turn induk** setelah spawn; hasil tiba di message
  berikutnya (`sessions-yield-tool.ts:1-8`). Ini beda dari OpenCode (yang tetap
  melayani) — OpenClaw **menutup giliran**.
- **Registry** (`subagent-registry.ts`): koordinasi registration → lifecycle →
  delivery → steering → recovery → persistence; ada `SUBAGENT_SUSPENDED_DELIVERY_HARD_CAP`
  (`subagent-registry.ts:42`) — cap hasil yang belum dikirim.
- `agents_wait` — tunggu sampai N run selesai (max 1000 ids, timeout).

**Yang diambil (pola)**: konsep child session terisolasi + registry persist +
wait/cap + delivery. OpenCode sudah lebih sederhana & terbukti untuk L1→L2;
OpenClaw berguna sebagai **referensi delivery/backlog guard** (cap hasil) yang
mungkin kita adopsi.

## 4. Multi-channel — Telegram & webhooks (L1 input)

- Telegram: `extensions/telegram/` — adapter channel; `channel.ts:1059 startAccount`,
  bot-core/bot-deps. **Terhubung ke core hanya lewat plugin-sdk**
  (`dispatchReplyWithBufferedBlockDispatcher` dari `openclaw/plugin-sdk/reply-dispatch-runtime`,
  `bot-deps.ts:20`) — channel tidak menyentuh loop agent langsung; ini pola
  **adapter terisolasi** yang bagus.
- Webhooks: `extensions/webhooks/` — `http.ts`, `config.ts` — HTTP inbound/outbound.
- Mapping channel→agent: `src/auto-reply/dispatch.ts:422 dispatchInboundMessage`
  → `reply/get-reply.ts:306 getReplyFromConfig` → `runEmbeddedAgent`.

**Yang diambil (pola)**: channel = adapter terpisah yang cuma kirim pesan ke core
(dispatch) dan terima delivery. Di arsitektur kita, "channel" = web + Telegram bot
kita sendiri yang memanggil API L1 (opencode serve). OpenClaw hanya jadi referensi
struktur extension, bukan kode yang diambil.

## 5. Memory lintas percakapan (kontinuitas identitas)

File: `src/claws/schema.ts:225`, `src/plugins/memory-state.ts`.

- `memory.search.rememberAcrossConversations: boolean` — wajib `true` kalau
  `sources` memuat `"sessions"` (schema.ts:235-242). Artinya agent bisa ingat
  percakapan sebelumnya dari sumber `memory`/`sessions`.
- Memory = plugin runtime (`plugins/memory-state.ts`): registry memoryCapabilities,
  corpus supplements, prompt supplements — memori diinjeksi ke prompt per-turn
  (`buildMemoryPromptSection`, `system-prompt.ts:322`). **Ini salah satu biang
  boros token** (memori 13k char di pengukuran).

**Yang diambil (konsep)**: persistence lintas percakapan. Di opencode, sudah ada
sesi SQLite (setiap sesi persist). Untuk identitas L1 lintas percakapan → cukup
simpan ringkasan (summary/notes) ke file/sesi terpisah, **tanpa injeksi penuh
per turn**.

## 6. Yang DIAMBIL vs DIBUANG (OpenClaw)

### Diambil (konsep/pola, bukan kode)
- Heartbeat config: `{every, activeHours}` + cooldown + skip-no-pending → ditiru
  ringkas atau via cron sistem.
- Cron/delivery-context: hasil job dikirim ke channel/target spesifik.
- Subagent registry: persist run, wait, cap delivery backlog.
- Channel-as-adapter: web/Telegram = adapter yang panggil API core.
- Memory: simpan ringkasan lintas percakapan, bukan injeksi penuh.

### Dibuang
- **Seluruh source OpenClaw** (408MB, 48 tool/turn) — tidak di-fork. TUI/GUI/UI
  (`apps/`, `ui/`) buang. Channel extension hanya referensi. Desktop/mobile buang.

## 7. Pemetaan ke arsitektur 3-level

| Level | Fungsi OpenClaw yang dijadikan pola |
|---|---|
| L1 Resepsionis | heartbeat `{every, activeHours}` + cooldown (bangun berkala utk nyapu pending/cek worker); cron untuk job; channel = adapter (Telegram/web sendiri → panggil API opencode); identitas lintas percakapan via ringkasan |
| L2 Build Agent | spawn child session terisolasi (mirip `spawnSubagentDirect`); registry run + wait + cap backlog; `sessions_yield` = referensi handoff end-turn |
| L3 Local Gate | tidak ada fungsi langsung dari OpenClaw utk gate/filter — tetap custom (lihat `arsitektur-gate.md` §3) |

## 8. Implikasi & pertanyaan terbuka

1. **Heartbeat token vs cron sistem**: OpenClaw pakai LLM wake per interval (mahal).
   Alternatif lebih murah untuk kita: cron sistem (systemd/crontab) yang memicu API
   L1 hanya saat ada kerjaan (mis. cek pending job 1×/menit, tanpa LLM). Belum
   diputuskan — perlu simulasi biaya.
2. **active-hours** berguna untuk batasi biaya (jangan bangun tengah malam) —
   pola ini layak diadopsi apa adanya.
3. **Delivery backlog cap** (OpenClaw `SUBAGENT_SUSPENDED_DELIVERY_HARD_CAP`) —
   apakah perlu di opencode? OpenCode pakai auto-notify (BackgroundJob) jadi
   backlog jarang; tapi cap tetap berguna kalau worker banyak.
4. **Aplikasi versi**: OpenClaw di vox = npm `2026.7.1-2` vs upstream `f870d93e`
   (lebih baru) — perilaku heartbeat/cron bisa beda versi.
