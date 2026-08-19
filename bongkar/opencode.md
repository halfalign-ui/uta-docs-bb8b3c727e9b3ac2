# OpenCode — Bongkar untuk Ekstraksi (Headless, Tanpa TUI/GUI)

> 2026-08-16. Commit `4643e65a` (branch dev), clone `/srv/dev/upstream/opencode`.
> Tujuan: ambil fungsi yang dibutuhkan arsitektur 3-level (`notes/arsitektur-gate.md`),
> **tanpa TUI/GUI**. Semua referensi `file:line` = commit tersebut.

## Ringkasan

OpenCode punya **server HTTP headless asli** (`opencode serve`) + **API REST lengkap**
untuk sesi/agent/event, plus **background subagent** (`Task {background:true}`).
Ini pas untuk Level 1 (resepsionis) dan Level 2 (build agent) arsitektur kita.
Yang diambil = `serve` + API route group tertentu. Yang dibuang = semua paket UI.

## 1. Server headless (`serve`)

File: `packages/opencode/src/cli/cmd/serve.ts` (27 baris), `server/server.ts` (226 baris).

- `serve` = "starts a headless opencode server" (`serve.ts:11`), **`instance: false`**
  (`serve.ts:13`) — server load instance per-request via header `x-opencode-directory`,
  jadi satu server bisa layani banyak direktori/proyek tanpa restart.
- Listener: `Server.listen(opts)` (`serve.ts:24`) → `NodeHttpServer` + `HttpRouter` +
  `HttpApiApp` (`server.ts:19-25`). Port/hostname dari args (`--port`, `--hostname`).
- **Auth wajib dipasang**: `OPENCODE_SERVER_PASSWORD` (`serve.ts:14-15`), Basic auth —
  `server/auth.ts` (username default `opencode`, `auth.ts:16`).
- Event source: `GET /event` = SSE (`groups/event.ts:14`).

**Diambil:** seluruh `serve` path. **Catatan:** jangan pakai `opencode web`
(`cli/cmd/web.ts`) — itu serve + buka browser GUI.

## 2. API route groups (REST) yang dibutuhkan

`packages/opencode/src/server/routes/instance/httpapi/` — `api.ts` menyusun
`RootHttpApi` dari `groups/*`.

### Kelompok sesi — `groups/session.ts` (inti L1+L2)

| Endpoint | Peran |
|---|---|
| `POST /session` (create) | buat sesi baru; payload `Session.CreateInput` (`session/session.ts:260-271`): `parentID`, `title`, `agent`, `model`, `metadata`, `permission`, `workspaceID` |
| `POST /session/:id/prompt` | kirim pesan, tunggu selesai (streaming response) |
| `POST /session/:id/promptAsync` | kirim pesan **tanpa menunggu** — cocok utk worker fire-and-forget (`groups/session.ts:329`) |
| `GET /session/:id/children` | list sesi anak (parent relation) — **monitoring worker** dari resepsionis (`handlers/session.ts:89-92`) |
| `GET /session/:id/messages` | riwayat pesan |
| `GET /session/:id/status` | status sesi (`groups/session.ts:121`) |
| `POST /session/:id/abort` | batalkan proses sesi (`handlers/session.ts:226-230`) |
| `POST /session/:id/fork` | fork sesi (buat worker dari titik tertentu) |
| `GET /session/:id/todo`, `diff` | todo list + diff — pantau progres build |

### Kelompok lain yang relevan

- `event.ts` — SSE stream event sesi (real-time monitoring, alternatif polling).
- `provider.ts` — resolusi model/provider (L1 cloud vs L3 local).
- `config.ts` — config runtime.
- `permission.ts` — permission/ruleset runtime.

**Diambil:** `session`, `event`, `provider`, `config`, `permission`. **Dibuang:**
`tui.ts` (TUI-specific), `control-plane`, `workspace`, `project-copy`, `pty`
(kecuali butuh terminal remote — TBD), `sync`, `mcp` (TBD), `question`
(TBD — dipakai approval flow).

## 3. Agent & model per-sesi (L1 cloud / L3 local)

File: `packages/opencode/src/agent/agent.ts` (453 baris).

- Definisi agent punya **opsional** `model: { providerID, modelID }` (`agent.ts:71`).
  Kalau kosong → pakai default/fallback.
- **Kunci untuk arsitektur kita**: saat spawn subagent via Task, model subagent =
  `next.model ?? { modelID: msg.info.modelID, providerID: msg.info.providerID }`
  (`tool/task.ts:181-185`) — artinya **tiap agent bisa didefinisikan dengan model
  sendiri** (resepsionis = cloud, build agent = cloud, gate = local) hanya lewat config.
- Agent default: `build` (default), `plan`, `general`, `explore`, `compaction`,
  `title`, `summary` (`agent.ts:141-215`). Semua bukan loop — loop ada di `session/`.
- **Custom agent via markdown**: `config/agent.ts` memindai `{agent,agents}/**/*.md`
  (frontmatter + prompt) → nama dari path (`config/agent.ts:20-33`). Jadi tinggal
  taruh `agent/build.md` dengan frontmatter `model:` untuk override.

**Diambil:** mekanisme custom agent + per-agent model. Ini yang bikin satu instalasi
bisa layani resepsionis (cloud) & gate (local) dengan config saja, tanpa kode baru.

## 4. Background subagent — pola ticket (L1→L2)

File: `packages/opencode/src/tool/task.ts`, `src/background/job.ts`,
`packages/core/src/background-job.ts`, `src/agent/subagent-permissions.ts`.

- Tool `Task` param `background: boolean` → "launches asynchronously and returns
  immediately" (`task.ts:26`), auto-notify saat selesai (`task.ts:29`), instruction
  "DO NOT sleep, poll, or proactively check on its progress" (`task.ts:60`).
- **Guard:** butuh `OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true` (`task.ts:98-101`);
  depth `subagent_depth` (default 1) (`task.ts:108-115`).
- Spawn: buat session anak (`sessions.create({ parentID, agent: next.name, ... })`,
  `task.ts:151-160`), lalu jalankan sebagai **BackgroundJob** (`task.ts:273-283`).
- BackgroundJob.Service (`core/background-job.ts:88-99`): `start`, `extend`,
  `wait`, `waitForPromotion`, `promote`, `cancel`. Status: `running | completed |
  error | cancelled` (`background-job.ts:7`). Semua dijalankan di scope Effect sendiri.
- Notify: `TaskTool.notifyBackgroundResult` menunggu job → injeksi hasil kembali ke
  sesi induk (`task.ts:245-283`).
- Permissions subagent: `deriveSubagentSessionPermission` (`subagent-permissions.ts`)
  — gabungkan deny rules induk + default deny `todowrite`/`task` kalau subagent
  belum izinkan. **Ini contoh guardrail yang bisa diadaptasi untuk gate L3.**

**Diambil:** pola background spawn + notify + permission inherit. Terbukti jalan
di vox-space (`notes/proof-of-run-vox-space.md`).

## 5. Turn loop & runtime LLM (tetap dipakai, tanpa UI)

File: `session/prompt.ts` (`ops()` + `loop()`), `session/processor.ts` (718 baris),
`session/llm/request.ts`, `llm/ai-sdk.ts` + `llm/native-request.ts`.

- Loop: `SessionPrompt.prompt` → `loop` (`prompt.ts:1052-1070`) → stream
  `llm.stream` → settle tool call → lanjut sampai `stop`/`compact`.
- Dua jalur runtime: ai-sdk (`session/llm/ai-sdk.ts`) & native
  (`native-request.ts`/`native-runtime.ts`) untuk provider tanpa SDK AI.
- Compact/summary: `compaction.ts`, `summary.ts` — session panjang tetap hemat token.
- **Semua ini murni server-side, tidak ada ketergantungan TUI.** UI hanya untuk
  menampilkan; loop jalan sendiri di `serve`.

**Diambil:** seluruh loop + compaction + LLM runtime apa adanya.

## 6. Storage SQLite (tetap dipakai)

File: `packages/core/src/database/database.ts`, `sqlite.node.ts`, `session/sql.ts`.

- DB: `~/.local/share/opencode/opencode.db` (override `OPENCODE_DB`),
  driver `node:sqlite` (DatabaseSync), WAL, busy_timeout (`database.ts:27-33`).
- Schema Drizzle snake_case: Session/Message/Part/Todo + V2 inbox (`sql.ts`).
- Model akses V1 (`Session.Service` → `MessageV2.page/stream`) sudah cukup untuk
  headless; jalur V2 ada tapi belum wajib (lihat pertanyaan terbuka map.md).

**Diambil:** `packages/core` database + `effect-sqlite-node`/`effect-drizzle-sqlite`.

## 7. Client SDK (untuk memanggil serve dari kode sendiri)

File: `packages/sdk/js/src/v2/client.ts` (93 baris), `packages/sdk/js/src/index.ts`.

- `createOpencodeClient(config)` — client TS kecil ke server HTTP (`v2/client.ts:50`).
- Contoh pemakaian nyata: `cli/cmd/run.ts` pakai `sdk.session.create` /
  `client.session.prompt` untuk mode non-interactive (`run.ts:519,859`).

**Diambil:** `packages/sdk` (kecil, berguna utk gate/tooling kita sendiri).

## 8. Yang DIAMBIL vs DIBUANG (ringkas)

### Diambil (headless core)
- `opencode` (serve, API routes: session/event/provider/config/permission, agent,
  session loop, tool system, background job, MCP client)
- `core` (database, session V2, model catalog, event manifest)
- `llm` (runtime LLM)
- `schema`, `protocol`, `function`/`plugin`
- `sdk` (client)
- `effect-sqlite-node`, `effect-drizzle-sqlite` (storage)

### Dibuang (TUI/GUI/enterprise)
- `tui`, `console`, `ui`, `web`, `desktop`, `app`, `session-ui`, `storybook`
- `server` (Node server untuk TUI/web — pakai `opencode serve` langsung),
  `control-plane`, `enterprise`, `slack`, `stats`
- `codemode`, `containers`, `identity`, `http-recorder`, `httpapi-codegen`
  → TBD (belum dipastikan kebutuhan).

## 9. Pemetaan ke arsitektur 3-level

| Level | Fungsi OpenCode yang diambil |
|---|---|
| L1 Resepsionis | `serve` + API session/event, `Task {background:true}` untuk ticket, `children` untuk monitor, custom agent `receptionist` (model cloud), SSE event |
| L2 Build Agent | custom agent `build` (model cloud), spawn sebagai background subagent session (parentID), `promptAsync` untuk fire-and-forget, todo/diff untuk progres |
| L3 Local Gate | **tidak dari OpenCode secara langsung** — OpenCode bisa jadi executor lokal (provider `local/qwen2.5-7b`, sudah dipakai vox-space), tapi **guardrail/filter custom** (arsitektur L3) belum ada di OpenCode. Yang bisa dipinjam: `subagent-permissions.ts`, `PermissionV1.Ruleset`, `truncate.ts` |

## 10. Implikasi & pertanyaan terbuka

1. **`x-opencode-directory`** — server bisa layani banyak direktori; berguna kalau
   tiap build agent diarahkan ke direktori kerja sendiri (isolasi). Perlu diverifikasi
   di versi 1.18.18 binary vox (cara kirim header).
2. **Jalur native runtime LLM** (`native-request.ts`) — kapan aktif untuk provider
   openai-compatible vs SDK-native (masih open question dari map.md).
3. **Guardrail L3 tetap custom** — OpenCode tidak punya "filter egress/ingress"
   untuk command dari agent eksternal; itu yang kita bangun sendiri.
4. **Aplikasi versi**: binary vox 1.18.18 vs source dev `4643e65a` — beberapa
   API (mis. `promptAsync`) perlu dicek ketersediaannya di 1.18.18 sebelum dipakai.
