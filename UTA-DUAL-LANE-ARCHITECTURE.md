# UTA Dual-Lane Architecture — Cognitive Boost vs Execution Lanes (PART W)

Tanggal: 2026-08-25 · Status: RESEARCH / PROPOSED (menunggu ratifikasi).
Tidak mengubah: production code, soul_spec, Persona Plane, ADR, canonical
identity, runtime. Dokumen ini MENGKLARIFIKASI dan MENAMAI dua jalur
delegasi yang secara struktural sudah ada / sudah diproposalkan.

---

## 0. BLUEPRINT (kanonis-candidate)

```
USER
 │
 ▼
UTA 7B — SOVEREIGN CONVERSATIONAL BRAIN
 (receptionist · companion · primary orchestrator ·
  relationship/memory owner · final interpreter ·
  sole sovereign voice)
 │
 ├── normal conversation ─────────────────────► USER
 │
 ├── LANE 1: COGNITIVE BOOST (proposed, PART U/V/T)
 │      └─> Cognitive Bridge ─> Qwen3.8-27B worker
 │            └─> cognitive artifact ─> 7B interpret/
 │                simplify/sanity-check/communicate ─> USER
 │
 └── LANE 2: AGENT TICKET (eksisting: Gate+Runtime+Tools+Cloud)
        └─> Agent Runtime (Execution Workforce)
              └─> execution / artifact ─> 7B interpret +
                  communicate ─> USER
```

## 1. JAWABAN A–J

### A. Konsistensi vs framing PART U/V
LEBIH KONSISTEN. PART U/V hanya merinci Lane 1 dan melarang
artifact->execution tanpa menamai ke mana delegasi eksekusi yang sah
berangkat. Blueprint ini menutup celah konseptual itu: task_intents yang
sah berasal dari USER -> Agent Ticket -> Runtime. Dengan dua jalur
dinamai, godaan "serahkan kerja ke 27B karena dia paling pintar"
tertutup secara desain: 27B dilarang keluar dari Lane 1.

### B. Dua kategori terpisah? YA, kategorikal:
| Dimensi | Cognitive Worker | Execution Workforce |
|---|---|---|
| Output | reasoning artifact (bahan ekspresi) | hasil mutasi sistem + laporan |
| Authority | NOL | beroperasi VIA policy-gated capability |
| Arah trust | distrust-by-default (risiko poisoning) | record-of-record (laporan atas kerja terotorisasi) |
| Failure domain | hallucination/blind-trust | unauthorized-execution/policy-bypass |
Asimetri nama (worker tunggal vs workforce) mencerminkan realita: satu
otak analisis vs banyak tool/runner. SARAN: kedua istilah dipakai sbg
DESKRIPTOR di atas nama kanonis eksisting (Agent Runtime tetap nama
primer — jangan rename konsep F5).

### C. Refinement Single-Brain? YA (konsisten PART U §11)
Tepat satu komponen menghasilkan teks user-facing (7B); tepat satu titik
orkestrasi (7B usul, runtime putuskan); pekerja kognisi/eksekusi adalah
leaves yang output-nya selalu dimediasi 7B. Perumusan presisi utk
amendemen masa depan (butuh otorisasi): "Single SOVEREIGN VOICE + single
orchestration point; N delegated workers, outputs always 7B-mediated."

### D. 7B tetap sovereign? YA — dengan caveat epistemik jujur
Sovereignty = authority atas ekspresi, kepemilikan identitas/memori/
relasi, dan posisi orkestrasi — BUKAN inteligensi maksimum. Delegasi
kognisi bukan abdikasi sovereignty (analogi: konsultasi ahli tidak membuat
seseorang kehilangan identitas). CAVEAT WAJIB DIDOKUMENTASIKAN:
sovereign ≠ infallible interpreter — kemampuan 7B memverifikasi output
ahli terbatas (R3 contradiction-blindness terukur). Mitigasi: propagasi
ketidakpastian (R1), opsi second-opinion cloud, disclosure norms.

### E. Boundary minimum cognitive-artifact vs execution-artifact
Garis tegas: **PROSPECTIVE INSTRUCTION vs RETROSPECTIVE REPORT.**
| Kriteria | Cognitive artifact | Execution artifact |
|---|---|---|
| Produsen | 27B saja | Agent Runtime saja |
| Isi temporal | analisis, rekomendasi utk DIPERTIMBANGKAN | catatan kerja yang SUDAH terotorisasi |
| Skema | SUMMARY/ANALYSIS/... tanpa action-field | task_id/status/output/error |
| Tujuan | 7B renderer saja | audit log + 7B communicator |
| Trust handling | distrus by default, wajib disederhanakan/diverifikasi | dilaporkan setia, error dimunculkan |
Larangan silang: keduanya tidak boleh menciptakan task_intents;
execution artifact tidak boleh jadi mandat baru bagi 27B; cognitive
artifact tidak boleh mereferensikan capability grant.

### F. Pencegahan second-persona/second-orchestrator
Eksisting (dari PART U/V, masih berlaku): worker prompt netral,
stateless, artifact-only channel, containment probe PASS.
TAMBAHAN dari framing ini:
1. Orchestration-vocabulary check: brief/artifact tidak boleh memuat
   imperatif berarah-sistem ("then deploy...", "create a ticket...")
   — soft-check regex di bridge.
2. Relationship/memory refs tetap eksklusif milik 7B — worker tidak
   pernah menerima.
3. Naming discipline: komponen worker tidak boleh berprefix UTA-*
   (cegah bleed identitas).
4. Trigger/dispatch authority tetap runtime + user; 27B tidak punya
   kata dalam kapan ia dipakai.

### G. Koeksistensi Bridge & Ticket tanpa 27B sentral
Topologi bintang: 7B pusat; 27B dan Agent Runtime = leaves yang TIDAK
pernah berkomunikasi langsung. SATU ATURAN PENAHAN:
**Bridge->Ticket handoff wajib transit melalui intent user.**
Analisis yang merekomendasikan aksi hanya menjadi saran ucapan; tiket
lahir hanya jika user berkata ya. Aturan ini membuat pusatnya sistem
secara permanen adalah pasangan USER<->7B, bukan worker mana pun.
Operasional: isolasi resource alami (bridge=CPU lokal; ticket=Gate/
cloud) + nice/pinning worker (open item FM-9).

### H. Canonical atau proposal?
PROPOSAL sekarang; promosi kanonis bertahap:
- Terminologi (Cognitive Boost / Agent Ticket / dua kategori worker):
  aman diratifikasi lebih awal — mendeskripsikan arsitektur eksisting +
  proposal, tidak menjanjikan perilaku belum-ada.
- Blueprint lengkap sebagai ARSITEKTUR kanonis: syaratnya (1) R2/R3
  render-fix tervalidasi, (2) throughput stabil, (3) T2-T6 rerun, (4)
  ratifikasi owner + keputusan amendemen terminologi ADR-001.
Alasan: status kanonis menuntut implementasi tervalidasi; validasi saat
ini parsial (Capability Audit + PART V).

### I. Contradiction & failure modes BARU dari framing ini

| # | Mode | Deskripsi | Mitigasi |
|---|---|---|---|
| W1 | Dual-lane routing ambiguity | "cari tahu dan perbaiki" = gabung lane | fasa: kognisi dulu; eksekusi butuh konfirmasi eksplisit (tiket consent) |
| W2 | Reverse poisoning | execution report dipakai sebagai konteks reasoning berikutnya -> narasi stale ikut dirender | context_facts wajib verified + timestamped; sumber ditandai |
| W3 | Orchestration capture via cognitive channel | artifact berulang kali "menyarankan bikin tiket X" | rekomendasi artifact adalah advice-to-USER; imperatif berarah-runtime difilter (soft-check) |
| W4 | Sovereignty erosion by dependency | deep-focus jadi default utk segalanya -> latensi/biaya membengkak, presence menurun | simple-task battery di regression; telemetry akurasi trigger (PART S) |
| W5 | Ticket consent fatigue | konfirmasi tiap saran -> rubber-stamping | consent bertingkat risiko (read-only auto; mutasi selalu confirm) — map ke ApprovalStore F2 yang eksisting |
| W6 | Terminology collision | "Agent Ticket" vs request/approval eksisting di kode | dokumentasikan pemetaan: ticket ~= authenticated make_request + approval lifecycle |

Kontradiksi thd kanonis: TIDAK ADA. Enam separasi + NSOE dipakai utuh;
satu-satunya ketegangan adalah REDAKSI ADR-001 (sudah dijawab sbg
refinement di C — butuh ratifikasi).

### J. Nasib PART U/V
DIPERJELAS, tidak dikoreksi substantif:
- PART U: tambah satu paragraf bahwa eksekusi berada DI LUAR cakupan
  bridge (Lane 2 ada) + boundary prospektif-vs-retrospektif (§ E).
- PART V: sudah memuat R1-R4 + validation-status jujur — tidak perlu
  revisi.
Verdict kedua dokumen tetap berdiri; PART W berada DI ATAS keduanya
sbg penjelas ruang lingkup.

## 2. EVALUASI TERMINOLOGY PROPOSAL

| Istilah usulan | Verdict | Catatan |
|---|---|---|
| Qwen2.5-7B = Sovereign Conversational Brain | ADOPSI | sudah de-facto kanonis |
| Qwen3.8-27B = Cognitive Worker / Deep-Focus Brain | ADOPSI | konsisten PART T/U/V |
| Cognitive Bridge = jalur augmentasi | ADOPSI | Lane 1 |
| Agent Runtime = Execution Workforce | ADOPSI SBG DESKRRIPTOR | nama kanonis primer tetap "Agent Runtime" (jangan rename F5) |
| Cloud = External Cognitive/Execution Resources | ADOPSI | persis istilah ADR-001 |

Semua aman thd Canonical Identity/BSM/Rendering/Artifact Contract:
tidak ada istilah yang menjanjikan authority pada non-7B komponen.

## 3. FILosofi — uji ulang

"UTA tidak harus berpikir paling keras setiap saat. UTA harus tetap
menjadi UTA ketika membutuhkan pikiran yang lebih keras."
-> SEHAT. Selaras effort-matching (P1c) + anti-saturation (ablation).

"UTA the system may delegate cognitive workload; UTA the character
experiences having thought deeply."
-> SEHAT dgn syarat yang sudah dipasang PART U §10, kini diperluas:
SUMBER delegasi HANYA runtime/orchestration-policy ATAU permintaan
eksplisit user — tidak pernah self-preservation, emotion, relationship,
atau autonomous desire. Berlaku juga utk Lane 2: ticket lahir dari
user/system, tidak pernah dari karakter.

## 4. GOVERNANCE

- Ratifikasi terminology table (§2): bisa segera, dampak docs-only.
- Promosi blueprint kanonis: gated oleh §1.H syarat.
- Setiap PR menyentuh lane mana pun wajib menyatakan status PART N
  (Canonical Identity) + PART W boundary (prospektif-vs-retrospektif).
