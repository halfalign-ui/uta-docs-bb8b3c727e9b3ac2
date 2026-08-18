# Matriks Ekstraksi Gabungan — Fungsi × Sumber × Level

> 2026-08-16. Setelah bongkar OpenCode, OpenClaw, dan Hermes (lihat masing-masing
> `<proyek>/notes/bongkar-headless.md`). Satu file rujukan: fungsi apa yang dibutuhkan
> arsitektur 3-level (`notes/arsitektur-gate.md`), diambil dari mana (`proyek:file:line`),
> dan cara ambilnya (code langsung / pola / konsep).

## Cara baca
- **code** = bisa dipakai apa adanya (bagian dari proyek yang di-fork/port).
- **pola** = logika/perilaku yang ditiru di implementasi kita (port manual).
- **konsep** = ide yang diadopsi ringkas (tanpa mengikuti implementasi sumber).
- **custom** = tidak ada di ketiga proyek, harus dibangun sendiri.

---

## Level 1 — Resepsionis (cloud AI, front-end)

| Fungsi | Sumber | Lokasi | Ambil |
|---|---|---|---|
| Server HTTP headless + REST API sesi | OpenCode | `packages/opencode/src/cli/cmd/serve.ts:11-24`, `server/server.ts:19-25`, route `instance/httpapi/groups/session.ts` | code |
| SSE event stream (real-time) | OpenCode | `groups/event.ts:14` (`GET /event`) | code |
| Custom agent + per-agent model | OpenCode | `agent/agent.ts:71` (`model:{providerID,modelID}`), `config/agent.ts:20-33` (agent markdown `{agent,agents}/**/*.md`) | code |
| Ticket: background subagent + auto-notify | OpenCode | `tool/task.ts:26,29` (`Task {background:true}`), `background/job.ts`, `core/background-job.ts:88-99` (start/extend/wait/promote/cancel) | code (terbukti) |
| Monitoring worker | OpenCode | `GET /session/:id/children` (`handlers/session.ts:89-92`), `status` (`groups/session.ts:121`) | code |
| Heartbeat/jadwal + active-hours + cooldown + skip-no-pending | OpenClaw | `src/claws/schema.ts:249` (`heartbeat:{every,activeHours}`), `heartbeat-runner-scheduler.ts:64`, `heartbeat-cooldown.ts`, `heartbeat-wake-policy.ts` | pola |
| Cron + delivery context | OpenClaw | `src/agents/tools/cron-tool.ts:1-8`, `src/cron/delivery-context*` | pola |
| Channel = adapter terisolasi | OpenClaw | `extensions/telegram/` (`bot-deps.ts:20` `dispatchReplyWithBufferedBlockDispatcher`), `extensions/webhooks/` | pola |
| Identitas lintas percakapan | OpenClaw | `src/plugins/memory-state.ts`, `schema.ts:225` — tapi **simpan ringkasan saja**, bukan injeksi penuh per turn | konsep |
| Referensi API web: runs/approval/steer/stop/SSE | Hermes | `gateway/platforms/api_server.py:5-24` (`/v1/runs/{id}/approval|steer|stop`, `/events`) | referensi |
| Approval queue per-session (web) | Hermes | `tools/approval.py:2689-2769` (`submit_pending`, `get_pending_gateway_approval`, `ack_gateway_approval`, `approve_session`) | pola |

## Level 2 — Build Agent (cloud AI, worker)

| Fungsi | Sumber | Lokasi | Ambil |
|---|---|---|---|
| Worker session terisolasi + parallel | OpenCode | spawn via Task (`tool/task.ts:151-160`), `POST /session` dengan `parentID` (`groups/session.ts:260-271`) | code |
| Fire-and-forget prompt | OpenCode | `POST /session/:id/promptAsync` (`groups/session.ts:329`) | code |
| Progres build (todo/diff) | OpenCode | `GET /session/:id/todo`, `diff` | code |
| Delegasi child + role `leaf`/`orchestrator` | Hermes | `tools/delegate_tool.py:3520-3522` | pola |
| Depth cap + `max_iterations` config-authoritative (model tidak bisa override) | Hermes | `delegate_tool.py:3547-3557` (depth default 2), `3561-3576` | pola |
| Concurrency cap (reject, bukan antre) | Hermes | `delegate_tool.py:793-812` (`max_concurrent_children`) | pola |
| Kontrol live: `list`/`steer`/`stop` (sinkron) | Hermes | `delegate_tool.py:213-303`, `3527-3532` | pola |
| Summary budget (hasil child dibatasi) | Hermes | `delegate_tool.py:981-990` | pola |
| Subagent blocked tools | Hermes | `delegate_tool.py:64-75` (`DELEGATE_BLOCKED_TOOLS`); OpenCode `subagent-permissions.ts` (deny default `todowrite`/`task`) | pola + code |
| Handoff end-turn (spawn lalu yield) | OpenClaw | `sessions-spawn-tool.ts:205` (swarm), `sessions-yield-tool.ts:1-8` | pola |
| Registry run + wait + cap backlog | OpenClaw | `subagent-registry.ts:42` (`SUBAGENT_SUSPENDED_DELIVERY_HARD_CAP`), `agents-wait-tool.ts` (max 1000 ids) | pola |

## Level 3 — Local Gate (local AI, eksekutor + guardrail)

| Fungsi | Sumber | Lokasi | Ambil |
|---|---|---|---|
| Eksekusi + process registry (spawn/poll/wait/kill) | Hermes | `tools/process_registry.py:17-29`, `ProcessSession` `:366-374` | code/pola |
| Isolasi proses worker (cgroup systemd) | Hermes | `process_registry.py:84-116` (`systemd-run --user --scope --unit=hermes-worker-<pid>`, memory dari `memory.max`) | pola (vox pakai systemd) |
| **Guard command berlapis (inti guardrail)** | Hermes | `tools/approval.py:4058` `check_all_command_guards` | pola (port ke TS) |
| Hardline floor unconditional (rm -rf /, mkfs, dd, shutdown, fork bomb) | Hermes | `approval.py:576` `detect_hardline_command`, blok di `:4080-4088` | pola |
| Sudo stdin guard (cegah tebak password) | Hermes | `approval.py:4090-4097` (`_check_sudo_stdin_guard`) | pola |
| User deny rules (blok sebelum yolo) | Hermes | `approval.py:4099-4105` (`_match_user_deny_rule`) | pola |
| Yolo session-scoped (bypass prompt, TIDAK bypass hardline/deny) | Hermes | `approval.py:4107-4110` | pola |
| Permanent allowlist | Hermes | `approval.py:2803` (`_command_matches_permanent_allowlist`) | pola |
| Cron auto-deny (tidak ada user saat itu) | Hermes | `approval.py:4117-4141` | pola |
| Content-level threat (homograph URL, pipe-to-interpreter, terminal injection) | Hermes | `tools/tirith_security.py` (`check_command_security`), dipanggil `approval.py:4143-4150` | pola |
| Skip container fast-path (isolasi sandbox) | Hermes | `approval.py:4070-4074` (`_should_skip_container_guards`) | pola |
| Permission inherit + deny default | OpenCode | `agent/subagent-permissions.ts` (`deriveSubagentSessionPermission`), `PermissionV1.Ruleset` | code |
| Output truncation | OpenCode | `truncate.ts` | code |
| Eksekutor model kecil | OpenCode (config) | provider `local/qwen2.5-7b` di vox; target 3B per desain | konfig |

---

## Pemetaan ke 6 guardrail wajib L3 (`arsitektur-gate.md` §3)

| # | Guardrail | Sumber pola | Status |
|---|---|---|---|
| 1 | Allowlist tools/commands | Hermes permanent allowlist (`approval.py:2803`); OpenCode `PermissionV1.Ruleset` | pola ada, kode custom |
| 2 | Filter output egress (secret/token/path/volume) | Hermes `_redact_terminal_error_text` (`tools/terminal_tool.py:57`); summary budget (`delegate_tool.py:981-990`); OpenCode `truncate.ts` | **perlu desain custom** (belum ada yang lengkap) |
| 3 | Filter input ingress (validasi command) | Hermes `check_all_command_guards` + tirith | port ke TS |
| 4 | Approval | Hermes approval queue per-session; OpenCode permission respond | port/adaptasi |
| 5 | Audit log | **tidak ada di 3 proyek** | custom |
| 6 | Resource limit (timeout/max output/no fork bomb) | Hermes `process_registry` timeout+memory+cgroup; OpenCode `truncate.ts` | port + custom |

## Protokol antar-level (kontrak JSON, `arsitektur-gate.md` §4)
- **Tiket L1→L2** — format custom; mekanisme terbukti (openCode background subagent + tiket ID sesi). Tidak ada proyek punya format tiket → custom.
- **Command L2→L3** — format custom. **ACP belum di-riset** (pertanyaan #4 `arsitektur-gate.md` §9).
- **Hasil L3→L2** — format custom + redact; OpenCode `truncate.ts` (volume), Hermes summary budget (isi).

## Ringkasan verdict per proyek
| Proyek | Dipakai sebagai | Verdict |
|---|---|---|
| **OpenCode** | Basis headless **L1+L2** | Fork/ambil code utuh: serve, API session/event, background job, agent, SDK |
| **Hermes** | Referensi **L3** (guardrail) + pola delegate **L2** + process registry + endpoint approval/steer (referensi L1) | Port/pola, **bukan** fork (~670k baris) |
| **OpenClaw** | Referensi konsep **L1** (heartbeat/cron/channel/registry/memory) | Konsep saja, tidak di-fork (408MB) |

## Keputusan tersisa sebelum implementasi (besar → kecil)
1. **Stack L3**: port pola Hermes (Python) ke TS agar satu bahasa dengan OpenCode? Atau jalankan Hermes/script Python sebagai gate service terpisah yang dipanggil L3?
2. **Protokol L2→L3**: riset **ACP** dulu (standar agent↔runtime) vs custom JSON via HTTP.
3. **L1+L2 satu instalasi opencode**: terbukti di vox; konfirmasi binary 1.18.18 mendukung `promptAsync` + `x-opencode-directory`.
4. **Heartbeat**: cron sistem (murah, memicu API L1) vs LLM wake ala OpenClaw (mahal).
5. **Model L3**: `qwen2.5-7b` (yang tersedia di vox) vs pasang 3B sesuai desain.
