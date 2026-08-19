# Hermes — Bongkar untuk Ekstraksi (Fokus L2+L3: Delegate, Eksekusi, Guardrail)

> 2026-08-16. Upstream `d5773bf`, clone `/srv/dev/upstream/hermes`.
> Tujuan: ambil fungsi yang dibutuhkan arsitektur 3-level (`notes/arsitektur-gate.md`),
> fokus **Level 2 (build agent)** = `delegate_task`, dan **Level 3 (local gate)** =
> process registry + approval/guardrail. Hermes TIDAK dipakai utuh (~670k baris py) —
> yang bernilai adalah **guardrail approval-nya** (paling lengkap di 3 proyek).
> Semua referensi `file:line` = commit `d5773bf`.

## Ringkasan

Hermes punya dua hal yang paling relevan untuk arsitektur kita:

1. **`delegate_task`** — spawn anak AIAgent terisolasi (single/batch/background),
   dengan kontrol (list/steer/stop), depth limit, concurrency cap, dan summary
   budget. (pola L2)
2. **Sistem approval/guardrail** (`tools/approval.py` ~5.000 baris) — filter
   command berbahaya, allowlist, deny-rule user, yolo mode, hardline block, approval
   via gateway/transport. **Ini kandidat terkuat untuk dipinjam di L3 gate.**

Ditambah: `process_registry.py` (manajemen proses background), `api_server.py`
(OpenAI-compatible HTTP — bisa jadi contoh antarmuka web L1).

## 1. delegate_task — delegasi L2 (worker terisolasi)

File: `tools/delegate_tool.py` (4.715+ baris), toolset `delegation` (`toolsets.py:299`).

- **Signature** (`delegate_tool.py:3428`): `goal`, `context`, `tasks[]`, `role`,
  `background`, `output_schema`, plus kontrol `action: list|steer|stop`.
- **Mode spawn**:
  - Single (`goal`), Batch (`tasks:[{goal,context,role}]`), dan **`background`**
    (async — "the chat is not blocked while they run", `delegate_tool.py:3500-3505`).
  - Batch background = **satu unit async**: semua child jalan di daemon executor,
    join semua, lalu **satu completion event** berisi hasil per-task.
- **Role** (`delegate_tool.py:3520-3522`): `leaf` (default, tidak bisa delegasi lagi)
  vs `orchestrator` (bisa spawn worker sendiri, bounded `max_spawn_depth`).
- **Guard & limits** (config `delegation.*`):
  - Depth: `delegation.max_spawn_depth` (default **2**), cek `parent_agent._delegate_depth`
    (`delegate_tool.py:3547-3557`).
  - Budget: `delegation.max_iterations` (default 50); **max_iterations dari model
    diabaikan** — config yang otoritatif (`delegate_tool.py:3561-3576`).
  - Concurrency: `delegation.max_concurrent_children` (cap background fan-out);
    saat penuh dispatch **ditolak, bukan diantre** (`delegate_tool.py:793-812`).
  - Kill switch: `set_spawn_paused` — TUI/RPC bisa freeze spawn baru tanpa
    mengganggu child yang berjalan (`delegate_tool.py:156-168`).
  - Kontrol live: `interrupt_subagent`, `steer_subagent` (masukkan course-correction
    ke child yang jalan), `list_active_subagents` (`delegate_tool.py:213-303`).
- **Isolasi child**: `AIAgent` baru + thread `ThreadPoolExecutor(initializer=_set_subagent_approval_cb)`
  (`delegate_tool.py:64-71`); fresh conversation, task_id sendiri, toolsets parent
  minus **DELEGATE_BLOCKED_TOOLS** (`delegate_tool.py:64-75`).
- **Summary budget**: hasil child dibatasi — hard ceiling + fraksi headroom parent
  (`delegate_tool.py:981-990`) supaya N child tidak meledakkan context induk.
- **Catatan**: `action=list|steer|stop` berjalan **sinkron** (tidak background)
  (`delegate_tool.py:3527-3532`).

**Yang diambil (pola)**: role leaf/orchestrator, depth cap, max_iterations
config-authoritative, concurrency reject (bukan queue), control actions
(steer/stop/list), summary budget. Ini persis kebutuhan L2.

## 2. Process registry — eksekusi & manajemen proses (L3)

File: `tools/process_registry.py`.

- API: `process_registry.spawn(env, cmd, task_id)`, `poll(id)`, `wait(id, timeout)`,
  `kill(id)` (`process_registry.py:17-29`). `ProcessSession` dataclass dengan Popen
  handle lokal (`process_registry.py:366-374`).
- **Isolasi cgroup**: spawn gateway-spawned local executor dibungkus
  `systemd-run --user --scope --unit=hermes-worker-<pid>` — worker di cgroup
  transien sendiri, jadi OOM worker tidak membunuh gateway (`process_registry.py:84-116`).
  Memory limit diambil dari `memory.max` cgroup induk, dibatasi aman.
- Ini contoh konkret **L3 execution**: spawn/kelola proses, timeout, kill, isolasi
  sumber daya.

**Yang diambil (pola)**: spawn + poll/wait/kill + timeout + cgroup isolation.
Implementasi kita bisa pakai `systemd-run` yang sama (vox-space pakai systemd).

## 3. Approval & guardrail — filter command berbahaya (L3, PALING BERHARGA)

File: `tools/approval.py` (~5.000 baris), dipakai `tools/terminal_tool.py:351`.

Alur guard (`check_all_command_guards`, `approval.py:4058`), berurutan:

1. **Skip container**: kalau backend container terisolasi & tanpa bind host →
   langsung approved (`approval.py:4070-4074`).
2. **Hardline floor** (`detect_hardline_command`, `approval.py:576`) — blok
   **unconditional**, tidak bisa di-bypass setting apa pun: `rm -rf /`, `mkfs`,
   `dd` ke raw device, shutdown/reboot, fork bomb, `kill -1`
   (`approval.py:4080-4088`).
3. **Sudo stdin guard** — `sudo -S` tanpa password terkonfigurasi = blok
   unconditional (cegah tebak password) (`approval.py:4090-4097`).
4. **User deny rules** (`approvals.deny` di config) — blok sebelum yolo
   (`approval.py:4099-4105`).
5. **Yolo bypass** (`--yolo` / `approvals.mode=off`) — session-scoped; melewati
   prompt tapi TIDAK melewati hardline/sudo/deny di atas.
6. **Permanent allowlist** (`_command_matches_permanent_allowlist`, `approval.py:2803`).
7. **Approval prompt** — `prompt_dangerous_approval` (`approval.py:2870`) via
   callback; konteks CLI vs gateway vs cron menentukan mode:
   - cron default `deny`: command berbahaya langsung diblokir (tidak ada user)
     (`approval.py:4117-4141`).
   - gateway: approval jadi **queue per-session** — `submit_pending`,
     `get_pending_gateway_approval`, `ack_gateway_approval`, `approve_session`
     (`approval.py:2689-2769`); user menjawab via API/transport, agent menunggu.
8. **Tirith** (`tools/tirith_security.py`) — deteksi konten-level: homograph URL,
   pipe-to-interpreter, terminal injection, dll. Dipanggil paralel di cron-deny
   (`approval.py:4143-4150`).

Pendukung lain: `path_security.py`, `spill_safety.py`, `url_safety.py`,
`write_approval.py` (guard per kategori).

**Yang diambil (pola + mungkin kode)**: struktur berlapis ini = blue print
guardrail L3 kita:
- hardline unconditional → allowlist → deny-rules → approval queue (gateway-style)
- cron auto-deny mode (tidak ada user saat itu)
- konteks per-session approval (bukan global)
- sinkronisasi via threading lock + contextvars (aman untuk worker thread)

## 4. api_server — OpenAI-compatible HTTP (referensi L1 web)

File: `gateway/platforms/api_server.py` (7.601 baris).

- Endpoint: `/v1/chat/completions`, `/v1/responses`, `/v1/models`,
  `/v1/capabilities`, `/api/sessions` CRUD, `/v1/runs` + `/events` (SSE) +
  `/approval` + `/steer` + `/stop`, `/health` (`api_server.py:5-24`).
- Sangat relevan: **`/v1/runs/{run_id}/approval`** — user resolve pending approval
  via HTTP; **`/steer`** — inject guidance ke agent jalan; **`/stop`** — interrupt.
  Ini persis mekanisme "user pantau & kontrol worker" yang kita mau di L1.
- Default port 8642; profile-aware (`/p/<profile>/v1/...`).

**Yang diambil (konsep)**: API shape untuk web L1 (chat, runs, approval, steer,
stop, SSE). OpenCode sudah punya versi TS-nya; Hermes memberi referensi endpoint
approval/steer yang belum ada di OpenCode.

## 5. Yang DIAMBIL vs DIBUANG (Hermes)

### Diambil (pola/konsep; beberapa kode)
- `delegate_task`: role, depth cap, budget config-authoritative, concurrency reject,
  steer/stop/list. (pola L2)
- `approval.py`: guard berlapis (hardline → allowlist → deny → queue → yolo),
  cron-deny, per-session approval, tirith. **(paling berharga — kandidat port ke L3)**
- `process_registry.py`: spawn/poll/wait/kill + cgroup isolation (via systemd-run).
- `api_server.py`: endpoint approval/steer/stop/SSE sebagai referensi web L1.

### Dibuang
- **Seluruh source Hermes** (~670k baris py, 9.316 file) tidak di-fork utuh.
- `apps/`, `web/`, `ui-tui/`, `tui_gateway/`, `website/`, `contributors/` → GUI/docs.
- 21/22 platform plugin, ~30/35 model-provider plugin, 7/8 memory plugin.
- Tools GUI/multimedia (browser, image/video, tts, computer_use, dsb).

## 6. Pemetaan ke arsitektur 3-level

| Level | Fungsi Hermes yang dijadikan pola |
|---|---|
| L1 Resepsionis | `api_server` endpoints (runs/approval/steer/stop/SSE) sbg referensi API web; session queue approval |
| L2 Build Agent | `delegate_task` role/depth/budget/concurrency/control (steer/stop/list); summary budget |
| L3 Local Gate | `approval.py` guard berlapis (hardline/allowlist/deny/queue/cron-deny) + `process_registry` (spawn/wait/kill + systemd cgroup) — **inti yang diambil dari Hermes** |

## 7. Implikasi & pertanyaan terbuka

1. **Port approval.py ke L3**: bahasa kita kemungkinan TS (bukan Python). Pola
   guard berlapisnya harus di-port manual ke TS/Node. Alternatif: jalankan Hermes
   sebagai "gate service" Python terpisah yang dipanggil L3? (perlu keputusan
   stack lintas bahasa).
2. **cron-deny default**: pola "cron job tanpa user = deny" sangat cocok untuk
   build agent background (tidak ada manusia saat command dijalankan) — tapi bisa
   memblokir kerjaan yang sah; perlu allowlist yang jelas.
3. **Approval queue per-session di web**: OpenCode punya `permissionRespond`
   (dilihat di bongkar opencode) — bandingkan mekanismenya dengan Hermes
   `submit_pending`/`ack_gateway_approval`; pilih yang lebih cocok untuk L1 web.
4. **systemd-run cgroup**: vox-space pakai systemd → pattern Hermes bisa langsung
   dipakai untuk isolasi worker lokal (L3).
