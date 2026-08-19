# Orkestrasi Multi-Agent — Riset terverifikasi (Fase 5)

Kebutuhan: **"resepsionis" selalu online** — menerima input (web/Telegram), lalu
menyerahkan build/task panjang ke **worker terpisah** (ticket-based), supaya
user tidak pernah terblokir oleh satu sesi yang sedang kerja lama.

Pertanyaan inti: mana dari 3 proyek yang secara native mendukung
"main agent tetap hidup + spawn worker async + notifikasi saat selesai"?

Hasil verifikasi dari source code (bukan klaim doc):

---

## 1. OpenCode — Background subagent (paling langsung cocok)

File kunci: `packages/opencode/src/tool/task.ts`, `src/background/job.ts`,
`src/core/background-job.ts`.

- Tool `Task` punya param `background: boolean`:
  - "launches the subagent asynchronously and returns immediately" (task.ts:26)
  - "You will be notified automatically when it finishes" (task.ts:29)
  - Guidance ke model: "DO NOT sleep, poll, or proactively check on its
    progress" (task.ts:60) — kontrak turn-based, bukan polling.
- **Requires** `OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true`,
  error eksplisit kalau tidak (task.ts:98-101).
- Background job dijalankan via `BackgroundJob.Service`:
  - `start`, `wait`, `extend`, `waitForPromotion`, `promote`, `cancel`
    (job.ts:34-45).
  - `notify` menunggu job selesai lalu menginjeksi hasilnya kembali ke sesi
    induk (task.ts:245-283) — induk "dibangunkan" saat worker selesai.
- Depth guard: `subagent_depth` (default 1) — maksimal 1 level nested
  (task.ts:108-115).
- Child session: `sessions.create({ parentID: ctx.sessionID, ... })` — worker
  adalah **session terpisah** dengan parent relation (task.ts:154-160),
  `task_id` bisa diresume (resume support, task.ts:139-143).
- Kesimpulan: **pola resepsionis = bawaan.** Main session (receptionist) pakai
  `Task` + `background:true` → kembali melayani user; build jalan sebagai
  subagent session; saat selesai induk di-notify. Model tunggal (yang sama
  dengan subagent `agent_type`), bukan spawn proses eksternal.

---

## 2. OpenClaw — Spawn + Yield (turn-end, bukan return)

File kunci: `src/agents/tools/sessions-spawn-tool.ts`,
`sessions-yield-tool.ts`, `subagents/registry/subagent-registry.ts`.

- `sessions_spawn`: mulai subagent/ACP session dengan inherited tool policy +
  delivery context (sessions-spawn-tool.ts:1-8). Ada mode swarm/parallel
  fan-out ("Swarm collector child for parallel fan-out; await via agents_wait",
  sessions-spawn-tool.ts:205).
- `sessions_yield`: **akhiri turn induk setelah spawn**, hasil tiba di message
  berikutnya ("Ends the current turn after subagent spawning so completion
  events can resume the session later", sessions-yield-tool.ts:1-8). Ini
  bukan "return & lanjut" seperti OpenCode — ini **turn-over**: resepsionis
  menutup gilirannya, di-resume nanti saat worker selesai.
- Delivery pressure: `subagentDeliveryBacklogPressure` — backlog hasil yang
  belum dikirim bisa memblokir spawn baru (sessions-spawn-tool.ts:409-413).
- Registry: `waitForSubagentCompletion(runId, timeout, entry, true)`
  (subagent-registry.ts:316).
- Kesimpulan: cocok untuk **orchestrator berbasis sesi multi-channel**
  (pendahulu Grok bot), tetapi mekanismenya "end turn & resume" — main agent
  tidak tetap melayani sambil worker jalan dalam giliran yang sama. Cocok
  untuk handoff, kurang pas untuk "tidak pernah berhenti merespon".

---

## 3. Hermes — delegate_task (in-process, parallel, paling kaya)

File kunci: `tools/delegate_tool.py`, `toolsets.py:299-409`.

- `delegate_task`: spawn child AIAgent **in-process** (ThreadPoolExecutor)
  dengan context terisolasi, inherited toolsets, terminal session sendiri,
  task_id sendiri (delegate_tool.py:4-15).
- Batch parallel: "Supports single-task and batch (parallel) modes"
  (delegate_tool.py:6-7).
- Kontrak konten: induk **hanya melihat delegation call + summary**, bukan
  tool-call/reasoning anak (delegate_tool.py:14-16).
- Safety: anak **tidak dapat** recursive delegation, `clarify`, menulis
  MEMORY.md bersama, `send_message` lintas platform, atau `cronjob`
  (DELEGATE_BLOCKED_TOOLS, delegate_tool.py:64-75).
- Capacity guard: `delegation.max_async_children` dihapus; sekarang ada
  batas async dispatch — saat penuh, dispatch baru **ditolak, bukan diantre**
  (delegate_tool.py:792-812).
- Model: top-level model calls jalan di background; orchestrator children
  menunggu worker-nya sendiri untuk sintesis hasil (delegate_tool.py:7-9).
- Kesimpulan: kaya & aman, tapi **worker = thread dalam proses yang sama** —
  bukan proses/session eksternal terpisah seperti OpenCode/OpenClaw. Cocok
  untuk fan-out riset/koding paralel dalam satu instance, bukan untuk
  "handoff ke mesin lain".

---

## Tabel perbandingan

| Aspek | OpenCode | OpenClaw | Hermes |
|---|---|---|---|
| API | `Task {background:true}` | `sessions_spawn` + `sessions_yield` | `delegate_task` |
| Main tetap melayani saat worker jalan | ✅ return immediately | ⚠️ end-turn & resume nanti | ✅ background async |
| Isolasi worker | session terpisah (parentID) | subagent session terpisah | thread in-process |
| Notifikasi selesai | auto-inject ke induk | resume via completion event | summary saat wait |
| Parallel | per-task (bisa banyak) | swarm fan-out via agents_wait | batch parallel |
| Guard | subagent_depth (default 1) | delivery backlog pressure | block async saat penuh |
| Env var | `OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true` | - | config `delegation.*` |
| Model | subagent_type (bisa berbeda) | subagent (bisa ACP) | model anak (bisa beda) |

---

## Rekomendasi untuk arsitektur kita

Pola **resepsionis + ticket worker** paling dekat dengan **OpenCode**
(`background:true` + session terpisah + auto-notify), dengan catatan:

1. Worker tetap satu proses server (bukan machine terpisah) — untuk
   "offload ke machine lain" butuh lapisan eksternal sendiri (mis. HTTP job).
2. `subagent_depth=1` berarti worker tidak bisa menelurkan worker lagi tanpa
   ubah config — perlu disesuaikan kalau mau tree of workers.
3. Kalau mau **multi-channel** (web + Telegram + WhatsApp), tetap pakai
   OpenCode sebagai *brain* + layer channel kita sendiri (vox) — konsisten
   dengan `basis-headless-server.md`.

Tahap selanjutnya (belum dieksekusi):
- Proof-of-run headless di vox: `opencode serve` + Telegram/web hook, uji
  `background:true` nyata.
- Ukur token/turn Hermes untuk memutuskan apakah layak sebagai alternatif
  full-agent (masih relevan untuk fan-out riset dalam satu instance).
