# Sumber & Versi

Catatan versi/commit yang jadi rujukan. Update tiap sinkronisasi.

## OpenClaw

| Sumber | Detail |
|---|---|
| npm vox | `2026.7.1-2` (`/usr/local/lib/node_modules/openclaw`) |
| **upstream (di codespace)** | `cef6e690d5573d06f3feef5fdf103906e842c618` — `test(auto-reply): drop persistence call count (#121742)` |
| upstream head (tercatat 2026-08-11) | `83dfd44eca487549d62516a0dd5dc297e5070092` (main, waktu riset awal) |
| **upstream clone analisis (2026-08-16)** | `f870d93e2a22bc25992f47eb03550d38620e6f4d` — `fix(ui): no-op the second reconcile of the same sessions.changed event (#124326)` |

Catatan: upstream main lebih baru dari npm vox. Kalau mau nyocokin versi vox → checkout tag `2026.7.1-2`.

## OpenCode

| Sumber | Detail |
|---|---|
| **upstream (di codespace)** | `3a90639cb57619a21e59f544b3e8d23ffed56f48` — `fix(ui): correct OC-2 weak icon color (#41504)` |
| upstream head (tercatat 2026-08-11) | `941e71dbbb94ea5b32226c2845585992dadb361f` (branch `dev`) |
| **upstream clone analisis (2026-08-16)** | `4643e65ad6334de3e4e68dedc201d5fbb828c9fe` — `fix(opencode): enable web search for Go (#42630)` (branch `dev`) |
| versi binary vox | opencode 1.18.5 (`/usr/local/bin/opencode`) |

## Hermes

| Sumber | Detail |
|---|---|
| **upstream (di codespace)** | `951ae62ffc51e2c279142905a054d0f696e2a54f` — `test(computer-use): pin 0.17+ split refs/content_refs merge behavior` (main, 2026-08-15) |
| **upstream clone analisis (2026-08-16)** | `d5773bfc3ad32148f0ff2e1de975fc94e37a0335` — `feat(desktop): Skills tab hub browser + full-skill detail pane; drop Browse Hub tab` (main) |
| repo | `NousResearch/hermes-agent` (MIT, Python, ~231k stars) |
| docs resmi | `hermes-agent.nousresearch.com/docs` — snapshot di `hermes/docs/` (`llms-full.md` 3.7MB + `llms-index.md`) |

## Codespace

| Item | Detail |
|---|---|
| codespace | `customai-dev-6vv6r44jqrx5hx4q7` (standardLinux32gb, 4-core/16GB/32GB) |
| tooling | node 22.23.2, pnpm 11.21.0, bun 1.3.14, go 1.24.13, git 2.51.1, gh |
| upstream path | `~/upstream/{openclaw,opencode,hermes}` |
| ssh access | `gh codespace ssh -c customai-dev-6vv6r44jqrx5hx4q7` |

## Fork kerja (arsitektur A — debloat)

| Item | Detail |
|---|---|
| repo | `halfalign-ui/CustomOpenClaw` (PRIVATE, fork openclaw/openclaw) |
| dibuat | 2026-08-11 (default branch only: `main`) |
| remote | `git remote add fork https://github.com/halfalign-ui/CustomOpenClaw.git` |
| baseline | sama dengan upstream `cef6e690` (main) |

Catatan: kerja debloat di fork ini; `~/upstream/openclaw` di codespace tetap read-only sebagai rujukan.

## Catatan sinkronisasi

- 2026-08-11: repo CustomAi dibuat (private, halfalign-ui). Codespace pertama gagal akses (butuh sshd) → tambah feature `sshd`, recreate. SSH jalan.
- 2026-08-11: upstream ter-clone di codespace; bun path via `~/.profile` (non-interactive shell nggak baca .bashrc).
- 2026-08-11: arsitektur final diputuskan = A (fork + trim). Fork `halfalign-ui/CustomOpenClaw` dibuat (private).
- 2026-08-16: repo direstruktur → folder per proyek (`openclaw/`, `opencode/`, `hermes/`), masing-masing `docs/` + `notes/`. Hermes ditambahkan (source + docs snapshot). `scripts/bootstrap.sh` + clone hermes. Akses `gh codespace` dibuka (scope `codespace`).
