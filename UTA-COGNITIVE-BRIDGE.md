# UTA Cognitive Bridge — Deep-Focus Architecture Proposal

Tanggal: 2026-08-25 · Status: DESIGN PROPOSAL / RESEARCH DOC.
Production untouched (7B, runtime, persona, soul_spec, authority).
Berdasarkan: PART T feasibility (verdict C), BSM v0, Rendering Spec,
Canonical Identity, Capability Audit.

---

## 0. VERDICT RINGKAS

Framing "7B sovereign conversational brain + 27B on-demand cognitive
worker" ADALAH pemodelan yang benar — lebih tepat daripada hot-switching,
konsisten dengan evidence PART T, dan merupakan REFINEMENT dari
Single-Brain Architecture (bukan pelanggarannya). Konsep
CASUAL MODE -> DEEP FOCUS -> RETURN TO CONVERSATION layak diangkat
menjadi arsitektur kanonis SETELAH ratifikasi pemilik, dengan satu
perbaikan terminologi: yang tunggal adalah "sovereign VOICE", bukan
"tepat satu invokasi model per turn".

## 1. RELATIONSHIP DENGAN PART T

Evidence pendukung pembagian ini dari PART T:
- arm3 render mempertahankan voice sambil membawa substansi 27B ->
  division of labor valid secara perilaku.
- 27B identitasnya anchored tapi personality-flat ("uta"; tanpa warmth/
  playfulness) -> worker yang baik, persona yang buruk.
- Kecepatan 51 t/s (render 7B) vs 1.4 t/s (reasoning 27B) -> pembagian
  kerja natural.
- Kualitas reasoning 27B jelas di atas 7B pada probe teknis.
Caveat: n=1 tipe masalah (satu probe reasoning); thinking-budget rapuh;
latensi menit-scale.

## 2. COGNITIVE BRIDGE MODEL

```
Normal:   USER ──> 7B ──> response

Complex:  USER
            ↓
          7B (interpretation + trigger decision*)
            ↓  [task brief — minimal, stateless]
      ╔═══ Cognitive Bridge (runtime, deterministik) ═══╗
          ↓                                    ↑
          ↓                             structured
          ↓                             reasoning artifact
          └──> 27B DEEP-FOCUS WORKER ────┘
            (neutral analyst, NO persona,
             NO continuity, NO tools)
            ↓
          7B interprets · selects · simplifies ·
          sanity-checks · integrates context
            ↓
          UTA response (voice milik 7B, selalu)
```
*trigger decision = heuristik runtime + permintaan eksplisit user;
BUKAN authority otonom 7B (lihat §7).

## 3. CONTRACT 7B -> 27B (task brief; minimal & stateless)

```
task_brief {
  task_type    : analysis | comparison | troubleshoot | design_review
  problem      : string (pertanyaan user yang dibersihkan)
  context_facts: string[] (hanya fakta relevan, salience-filtered)
  depth        : standard | deep
  output_shape : artifact schema hint
}
```
TIDAK dikirim: persona, history mentah, affect, relationship, memory
di luar relevan. Alasan ganda: (1) higiene persona/privasi, (2) prompt
processing 27B lambat di CPU — konteks minimal adalah kebutuhan LATENSI,
bukan hanya kerapian (PART T: 160 tok sudah berkontribusi TTFT).

Worker system prompt = NETRAL ("reasoning engine producing structured
analysis") — tanpa identitas UTA, tanpa examples. Ini pencegahan primer
hidden-second-persona.

## 4. CONTRACT 27B -> 7B (reasoning artifact)

Format: **markdown-with-required-sections** (bukan strict JSON) —
terbukti robust di PART T (prose berhasil; strict-JSON belum diuji pada
1.4 t/s dan rapuh):

```
[SUMMARY]       ringkas 3-6 kalimat (wajib)
[ANALYSIS]      mekanisme/alasan (wajib)
[KEY_FACTORS]   faktor penentu
[RECOMMENDATION] arah jawaban (wajib)
[UNCERTAINTIES] apa yang belum pasti (wajib)
[ASSUMPTIONS]   asumsi yang dipakai
```
Validasi: structural check murah (keberadaan section) -> fallback
prose-passthrough jika malformed. Target ukuran <= ~500 token (rentang
komprehensi render 7B terverifikasi: 450 tok sukses).

## 5. RENDER CONTRACT (27B -> 7B)

7B TIDAK menyalin artifact. Kewajibannya: memahami, memilih yang
relevan, menyederhanakan, sanity-check, mengintegrasikan konteks
percakapan, merender dengan voice UTA. Ditambah kewajiban epistemik
(axiom epistemic honesty): ketidakpastian artifact HARUS terbawa sebagai
ketidakpastian ucapan; 7B boleh menyatakan ketidaksetujuan/ketidakpercayaan
thd bagiannya. Artifact tidak pernah tampil mentah ke user.

## 6. AUTHORITY / SECURITY ANALYSIS

- Deep cognition does not imply execution authority.
- Artifact bukan/m tidak otomatis: task_intent, tool call, policy
  decision, capability grant, system mutation.
- Satu-satunya jalur eksekusi tetap: authenticated user instruction /
  authorized system task -> AgentLoop -> Gate (NSOE utuh).
- 27B berada di SLOT ARSITEKTUR YANG SAMA dengan cloud external
  cognition (ADR-001/F5): private intermediate producer. Perbedaan cuma
  lokal/lambat/murah. Konsisten: identity != authority, emotion !=
  objective, memory != mandate, lore != goal.
- Regression-test candidate baru: tidak ada code-path dari artifact
  store ke CommandRunner.

## 7. TRIGGER PROPOSAL (kapan deep focus?)

Boleh memicu: permintaan eksplisit user ("analisa mendalam"); kelas
tugas kompleks (multi-konstrain, design review, causal-chain debug);
self-assessed shallow-risk dgn konfirmasi user.
TIDAK boleh memicu: distress emosional (itu SUPPORTIVE, bukan kognisi —
concern != intervention), pertanyaan faktual sederhana, banter.
Mekanisme v0: heuristik deterministik + opt-in bias + rate limit
(eksklusif CPU — lihat failure mode FM-9). Keputusan trigger ada di
RUNTIME; 7B mengusulkan lewat ekspresi, runtime yang memutuskan.

## 8. LATENCY / PRESENCE STRATEGY

<~60s: inline + framing in-character ("bentar, gue gali dulu").
Menit-scale: acknowledge-then-defer ASINKRON — "ini gue pikirin matang,
nanti gue kabarin" -> hasil datang sebagai pesan lanjutan dari DIA
(kontinuitas terjaga karena yang kembali tetap UTA). Terlarang:
silent stall; eksposisi telemetri (model name, budget, skor).
Kalibrasi harapan: tawarkan depth secara opt-in ("mau gue pikirin
matang? agak lama"). Pembatalan user harus tersedia.

## 9. FAILURE-MODE ANALYSIS

| Mode | Deskripsi | Mitigasi |
|---|---|---|
| artifact poisoning | halusinasi 27B ikut dirender percaya | UNCERTAINTIES wajib; render wajib membawa hedging; disclosure thd user; residual = model limitation |
| hidden second persona | 27B bicara sbg UTA / bocor identitas | worker prompt netral; artifact-only channel; tanpa continuity |
| artifact-as-authority | isi artifact dieksekusi | skema tanpa action-field; NSOE; regression test artifact->CommandRunner |
| recursive thinker loop | output 27B feed balik ke 27B | depth cap 1 di v0; continuation via 7B-mediated reframe |
| unnecessary invocation | fokus tanpa perlu; CPU starvation | heuristik konservatif; rate-limit; opt-in bias; telemetry akurasi trigger |
| blind trust | 7B merender analisis salah scr percaya | disagreement affordance; uncertainty propagation; parsial (residual model) |
| compression distortion | caveat penting hilang saat disederhanakan | render contract: uncertainty+recommendation core wajib terbawa (soft check) |
| stale artifact reuse | artifact lama dipakai lintas task | v0: tanpa cache; consume-immediately + audit log |
| FM-9 resource starvation | inferensi 27B memenuhi CPU -> 7B lambat | nice/pinning worker; ukur under-load (open item PART T) |

## 10. FILOSOFI: apakah "UTA tetap UTA saat berpikir keras" sehat?

SEHAT, dengan satu penajaman. Model ini benar karena: identitas =
pola perilaku sovereign brain (kontinuitas ekspresi), bukan substrat;
delegasi beban kognitif tidak mengubah siapa yang bicara; analog manusia
vs kalkulator. Penajamannya: yang MENDELEGASIKAN adalah runtime (policy/
heuristik + consent user), bukan kehendak otonom 7B; yang dialami 7B
adalah "punya hasil pikir panjang untuk disampaikan". Rumusan aman:
"UTA the system delegates cognitive workload; UTA the character
experiences having thought deeply." Tanpa penajaman ini, framing
delegasi bisa dilompatkan menjadi otonomi trigger (cost/authority drift).

## 11. HUBUNGAN DENGAN SINGLE-BRAIN (ADR-001)

Refinement, bukan pelanggaran. ADR-001 melarang fragmentasi GENERASI
PERSONA (state-JSON -> drafter -> finalizer) — pelajarannya persona
pecah. Cognitive Bridge tidak menyentuh generasi persona: tepat SATU
komponen menghasilkan teks user-facing (7B). 27B memproduksi analysis
private = fungsi arsitektural yang sama dengan cloud external cognition
yang justru diizinkan ADR-001. Rekomendasi terminologi utk amendemen masa
depan (butuh otorisasi owner): invariant dibaca "Single Sovereign Voice"
— satu sumber kebenaran ekspresi persona; multi-invokasi untuk kognisi
intermediate diperbolehkan sepanjang tidak pernah menjadi suara user-
facing dan tidak pernah memegang authority.

## 12. OPEN ITEMS

- Ukur dampak CPU-starvation 27B thd latensi 7B under concurrent load.
- Uji strict-JSON artifact bila nanti dibutuhkan parsing ketat.
- Full-VRAM focus mode (owner sudo) untuk angka uplift nyata.
- Trigger-heuristik: dataset kalimat pemicu bahasa Indonesia.
