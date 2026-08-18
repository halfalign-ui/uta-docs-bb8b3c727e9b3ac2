# UTA — ARCHITECTURE DESIGN REPORT

> Status: DESIGN — proposal saja. Tidak ada komponen yang diimplementasikan.
> Referensi: `blueprint-uta-v1.md` (UTA) + discovery `UTA_PROJECT_STATE.md`.
> Tanggal: 2026-08-16.

## Bagian 0 — Ekstraksi Blueprint (sesuai permintaan)

1. **Architectural layers** — 3 level, tumpukan orkestrasi:
   `User → L1 Resepsionis (cloud AI) → L2 Build Agent (cloud AI) → L3 Local Gate (local AI)`.
2. **Local Gate responsibilities** — satu-satunya akses ke perangkat; eksekusi
   command/script; **6 guardrail wajib** (allowlist tools, filter egress, filter
   ingress, approval, audit log, resource limit); redact/truncate output.
3. **Cloud/local boundary** — cloud AI **tidak pernah** mengakses device langsung.
   Semua eksekusi lewat Local Gate. Local Gate = security boundary.
4. **Vault/private workspace** — **TIDAK disebut blueprint**.
   → **UNKNOWN / REQUIRES DESIGN DECISION** (diusulkan di Bagian D).
5. **MCP/tooling requirements** — blueprint mengecualikan MCP untuk otomasi web
   (pelanggaran ToS). MCP untuk domain lokal (filesystem/vault/workspace/system)
   **tidak dilarang** → dipakai sebagai tool-dispatch di Local Gate (Bagian C).
6. **Approval requirements** — operasi berbahaya butuh approval; deny-default saat
   tidak ada user (cron/background); allowlist + deny rules.
7. **Audit requirements** — audit log wajib (guardrail #5); protokol JSON
   antar-level dirancang agar auditable.
8. **Trust boundaries** — cloud = untrusted untuk akses langsung; LLM (lokal
   maupun cloud) **bukan** security boundary; policy deterministik di luar model.
9. **Assumptions eksplisit blueprint** — L1/L2 pakai model cloud berkualitas; L3
   pakai model lokal kecil (7B kandidat, **target 1.5–3B**); protokol JSON; ambil
   **pola** open source, bukan fork; tanpa otomasi web/browser.
10. **Unresolved design questions** (dari blueprint §10):
    (a) kecukupan model kecil GPT-class untuk L3; (b) stack L3 TS vs Python;
    (c) protokol L2→L3 custom vs ACP; (d) heartbeat L1 (AI vs cron); (e) filter
    egress cukup atau perlu sandbox penuh; (f) channel L1 (web vs Telegram).

## Bagian A — Local Gate (komponen inti)

**Rekomendasi:** service broker eksekusi (proses terpisah, bukan plugin di dalam
LLM), dengan **policy engine deterministik** sebagai inti.

```
CLOUD (L1/L2) ──▶ Ingress API (auth + validasi skema kontrak)
                          │
                          ▼
                  POLICY ENGINE (deterministik, tanpa LLM)
                  ├─ allowlist tools / deny rules
                  ├─ scope per tiket (paths, hosts, services)
                  ├─ approval gate (user inline / deny-default)
                  └─ ingress filter (normalisasi command)
                          │
                          ▼
                   TOOL DISPATCH
                   ├─ MCP: filesystem / workspace / vault / system
                   ├─ command runner (isolasi, timeout, limit)
                   └─ service manager (systemd, approval)
                          │
                          ▼
                  OUTPUT PIPELINE
                  ├─ egress filter (redact secret, batas volume)
                  └─ AUDIT LOG (semua kejadian)
                          │
                          ▼
CLOUD (L1/L2) ◀── hasil (teredact) + audit id
```

Keputusan desain:
- **Policy sebelum model.** Semua keputusan berbahaya ditentukan aturan
  deterministik; model (cloud/local) hanya berproduksi *proposal* command.
- **Tidak ada mode "full trust".** Setiap command masuk harus lolos schema + allowlist.
- **Bahasa**: Python 3 diusulkan (L3 butuh ekosistem policy/audit cepat & portabel,
  dan service gateway lokal sudah Python); **keputusan TS-vs-Python tetap terbuka**
  (lihat Unresolved). Tidak ada pilihan dipaksa sebelum persetujuan.

## Bagian B — Orchestrator (model lokal kecil)

**Apakah model lokal kecil diperlukan? Ya — sebagai komponen orkestrasi.**

| Atribut | Usulan |
|---|---|
| Parameter class | **1.5–3B** (target; tidak perlu ≥7B) |
| Konteks | 4–8K token cukup (bukan analisis kode besar) |
| Concurrency | 1–2 request (single user; bisa antre) |
| Kebutuhan VRAM/RAM | ~2 GiB VRAM (Q4, GPU) atau ~4 GiB RAM (CPU) |
| Peran | intent classification, routing (local-vs-cloud), tool selection, MCP dispatch, struktur command, validasi, recovery sederhana, interpretasi hasil |
| Kenapa perlu | (1) routing & klasifikasi harus jalan **offline** walau cloud mati; (2) konteks pribadi tidak selalu boleh ke cloud; (3) meringankan L3 jadi eksekutor murni |
| Batas | Bukan penulis kode; reasoning berat tetap di cloud |

Catatan VRAM: mesin saat ini mengalokasi **6.2 GiB** VRAM untuk 7B yang sudah
berjalan. Model 1.5–3B memakai jauh lebih sedikit; jika dipakai berdampingan
maupun menggantikan, sisa VRAM cukup. Keputusan "ganti vs paralel" = Unresolved.

## Bagian C — MCP Boundaries

Server MCP diusulkan (tidak diinstal) dengan scope sempit, satu server per domain:

| MCP Server | Capability | Akses |
|---|---|---|
| `filesystem` | baca/tulis workspace proyek saja | workspace yang diizinkan per tiket |
| `workspace` | manajemen proyek (create/list/open) | hanya /data/workspaces |
| `vault` | baca/tulis data pribadi **via policy**, bukan raw fs | approval-gated, read-only default |
| `system` | info sistem + manajemen service (systemd) | read-only; start/stop butuh approval |

Aturan: **deny-by-default**; setiap server hanya lihat subset path yang disetujui;
tidak ada MCP untuk web/browser (sesuai blueprint §8).

## Bagian D — Vault (private storage)

**UNKNOWN di blueprint → diusulkan di sini.**

Isolasi diusulkan: **dedicated service account + filesystem permissions +
systemd sandboxing**, dengan `vault` MCP sebagai satu-satunya interface AI.

Perbandingan teknologi:

| Opsi | Kuat | Lemah | Rekomendasi |
|---|---|---|---|
| LXC | isolasi penuh | daemon inactive; perlu bikin & maintain; snapshot kompleks | Tahap lanjut |
| Docker | cepat, familiar | overhead image; daemon root; attack surface besar; resource | Hanya bila perlu |
| systemd sandboxing (DynamicUser, ReadOnlyPaths, PrivateMounts) | ringan, bawaan distro, zero infra | isolasi per-proses, bukan per-kernel | **Utama (fase awal)** |
| Dedicated user + perms | sangat ringan, paling mudah diaudit | tidak isolasi kernel | **Utama (fase awal)** |
| Kombinasi di atas | — | — | Pola akhir |

- Lokasi usulan vault: **`/data/vault`** (berdekatan dengan data pribadi, terpisah
  dari user session `/home/vox`), owned `vault` user (baru), mode 0700.
- Prinsip: *"AI hanya menerima capability yang eksplisit diberi"* — vault
  read-only default, write hanya via approval, egress ke cloud selalu teredact.
- `/data/win11.iso` dan `/data/models` **bukan** vault; keduanya aset yang
  diputuskan terpisah.

## Bagian E — Security Model

| Aspek | Desain |
|---|---|
| Trust boundaries | Cloud L1/L2 = UNTRUSTED utk akses device. Local Gate + policy engine = TRUST boundary. LLM (lokal) ≠ boundary. Vault user + perms = data boundary |
| Capabilities | deny-by-default; allowlist tools; scope per tiket (path/host/service) |
| Izin R/W | Gate: baca workspace → redact; tulis hanya ke path scope; vault read-only default; system read-only default |
| Approval gates | destructive/network/service/vault-write → approval; background/cron → deny-default |
| Secret handling | tidak pernah ke cloud (egress filter + secret redaction); secret lokal via env/file dengan perms ketat, bukan di command log |
| Audit logging | semua command/hasil/approval → `vault` audit log (append-only) + duplikat ke `/secondary/logs` |

Konfigurasi keamanan yang harus dibereskan di fase implementasi (bukan sekarang):
UFW masih inactive, sudo NOPASSWD penuh, secrets di file biasa.

## Bagian F — Resource Budget (perkiraan)

| Komponen | RAM | CPU | VRAM | Storage |
|---|---|---|---|---|
| Local Gate (policy+audit+runner) | ~512 MiB | 0.5 vCPU | 0 | 0.2 GiB (kode+audit) |
| MCP servers (3–4) | ~256 MiB | 0.2 vCPU | 0 | ~0.1 GiB |
| Vault (fs + snapshot ringan) | ~128 MiB | ~0 | 0 | sesuai isi pribadi |
| Local inference 1.5–3B | 0.5–1 GiB | 1 vCPU | **~2 GiB** | 1–2 GiB (GGUF Q4) |
| Monitoring (journal/health) | ~256 MiB | 0.1 vCPU | 0 | logs → /secondary/logs |
| **Total tambahan** | **~1.7 GiB** | ~2 vCPU | **~2 GiB** | ~4 GiB |

Kondisi saat ini: RAM available 12 GiB, vCPU 12 idle (skala 24%), VRAM 8 GiB
(6.2 GiB terpakai 7B), free storage 678 GiB di `/data` → **budget muat**.

## Keputusan yang Belum Diselesaikan (UNRESOLVED)

1. Stack L3: Python (diusulkan) vs TS — perlu keputusan setelah evaluasi.
2. Protokol L2→L3: custom JSON via HTTP vs **ACP** (Agent Client Protocol).
3. Model kecil spesifik (1.5–3B mana) & apakah **ganti** atau **paralel** dgn 7B.
4. Vault: konfirmasi lokasi `/data/vault` + teknologi final (systemd sandbox + user).
5. Channel L1: web dulu vs Telegram dulu.
6. Heartbeat L1: AI bangun berkala vs cron sistem.
7. Apakah instance opencode serve (3456) bagian dari L3/Local Gate atau bukan.
8. Kebijakan firewall & sudo (hardening) yang akan disetujui di fase implementasi.

## Urutan Implementasi yang Diusulkan

1. **F1 — Desain final disetujui** (resolusi Unresolved #1–#3).
2. **F2 — Local Gate core** tanpa model: policy engine + kontrak JSON + audit +
   command runner terisolasi. (basis deterministik)
3. **F3 — Guardrail deterministik**: allowlist, ingress/egress filter, approval.
4. **F4 — MCP boundaries**: `filesystem` + `workspace` dulu; `system`; `vault` terakhir.
5. **F5 — Orchestrator model kecil** (1.5–3B) untuk routing/klasifikasi.
6. **F6 — Integrasi cloud L1/L2**: protokol tiket/command/hasil + channel L1.
7. **F7 — Vault**: dedicated user, `/data/vault`, approval write, secret handling.
8. **F8 — Hardening**: UFW, sudo minimal, auth server, backup, monitoring.

> **Setiap fase menunggu approval eksplisit.** Fase ini (discovery+design) selesai.
