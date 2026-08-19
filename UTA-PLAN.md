# UTA — PROJECT PLAN (FINAL)

> Diperbarui setelah F7. Referensi: `UTA_ARCHITECTURE_FINAL.md`.

## Prinsip

- Mesin ini = Local Gate / orchestration node, **bukan** agent coding otonom.
- Reasoning berat di cloud; lokal = eksekusi + policy + audit.
- LLM bukan security boundary; policy deterministik yang memutuskan.
- Deterministic-first: model lokal opsional, bukan prerequisite.

## Fase Implementasi

### F0 — Discovery ✅
### F1 — Desain Final ✅
### F2 — Local Gate Core + Guardrail ✅
- 70 test OK. Policy engine, ingress/egress filter, approval gate, audit log.

### F3 — MCP Tool Boundary ✅
- +49 = 119 test OK. 3 MCP servers, 11 tools, scope enforcement.

### F4 — Vault & Secret Handling ✅
- +29 = 148 test OK. Vault MCP server (5 tools), path containment, audit redaction.

### F5 — Cloud Execution & Agent Runtime ✅
- +61 = 209 test OK. Cloud module (12→13 files), 5 MCP tools, endpoint validation, quota, lifecycle.

### F5A — Cloud Runtime Operational Completion ✅
- +55 = 264 test OK. RuntimeIsolation (dirs, sanitization, cleanup), fixed fallback, streaming tests, security regression.
- Commit: `e0f0a7c`. See `F5A-REPORT.md`.

### F6 — Heartbeat & Background ✅
- +40 = **304 test OK**. HeartbeatRunner (retry, audit, policy), 4 tasks, systemd timer/oneshot, CLI entry.
- Commits: `686f8b1`, `d388ff5`. See `F6-REPORT.md`.
- Deployed: systemd timer active on vox-space, 5-min interval.

### F7 — Local Intelligence Runtime + Tool-Calling Substrate ✅
- P0 Architecture: **COMPLETE**. See `F7-P0-ARCHITECTURE.md`.
- P1 Model Interface: **COMPLETE**. See `F7-P1-REPORT.md`.
- P2 Agent Loop Integration: **COMPLETE**. See `F7-P2-REPORT.md`.
- P3 Local Runtime: **COMPLETE**. See `F7-P3-REPORT.md`.
- P4 Capability Benchmark: **COMPLETE**. See `F7-P4-REPORT.md`.
  - Result: 11 PASS, 1 PARTIAL, 1 FAIL
  - Verdict: **LOCAL MODEL USABLE WITH LIMITATIONS**
- **Test Baseline**: 457 test OK (0 regression).

### F8 — Hardening & Deploy (ACTIVE)

## Status Saat Ini

- F1 DESIGN: COMPLETE
- F2 (Local Gate core): **COMPLETE** — 70 test OK
- F3 (MCP tool boundary): **COMPLETE** — 119 test OK
- F4 (Vault & secret): **COMPLETE** — 148 test OK
- F5 (Cloud execution): **COMPLETE** — 209 test OK
- F5A (Runtime isolation): **COMPLETE** — 264 test OK
- F6 (Heartbeat & background): **COMPLETE** — 304 test OK
- F7 (Local Intelligence Runtime): **COMPLETE** — 457 test OK
- SYSTEM CHANGES: vault user + /data/vault dir (vox-space)
- INSTALLATIONS: mcp==2.0.0 (Python 3.14.4 on vox-space)
- PRODUCTION SERVICES: uta-heartbeat.timer deployed and active (vox-space)
- F7 CLOSED. Proceeding to F8 — Hardening & Deploy.
