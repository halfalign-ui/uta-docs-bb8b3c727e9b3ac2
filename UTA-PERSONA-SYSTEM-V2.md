# UTA Persona System V2 — Design / Research Proposal

Tanggal: 2026-08-25
Status: **PROPOSAL RISET — BUKAN IMPLEMENTASI PRODUKSI.**
Production Persona Plane (soul_spec.json v1.0) dan ADR-001 tetap FROZEN.
Dokumen ini mendefinisikan model konseptual + hipotesis yang layak diuji,
berdasarkan PERSONA-RESEARCH-FINDINGS.md.

---

## 0. Invariant yang TIDAK boleh dilanggar

Setiap ide dalam proposal ini tunduk pada:

```
USER
  ↓
UTA Runtime / Context Assembly        ← satu-satunya tempat logika baru
  ↓
Qwen 2.5-7B (Sovereign Conversational Brain)
  ↓
response
```

- **Single-Brain invariant**: tepat SATU generative call per turn.
  TIDAK ADA Cognitive JSON → Draft → Finalizer.
  TIDAK ADA second generative persona model.
  TIDAK ADA fragmentasi generation (pelajaran ADR-001).
- Runtime authority absolut; cloud = external cognitive resources,
  BUKAN persona engine.
- Model tetap sumber voice & prosody; lapisan lain hanya MEMBENTUK
  KONTEKS yang dilihat otak — tidak pernah menyunting output.

Semua yang diproposalkan di bawah adalah *context assembly* logic —
bukan pipeline generation.

## 1. Diagram konseptual

```
                UTA Persona System (V2 concept)
                       │
        ┌──────────────┼──────────────┐
        │              │              │
     Identity       Behavior        State          ← lapisan persona
        │              │              │               (representasi
        └──────────────┼──────────────┘                terpisah, lokasi
                       │                               bisa berbeda)
                    Memory                             ← continuity store
                       │
                  Relationship                           ← view ke user
                       │
                    Context                             ← window live
                       │
                 ╔═══════════════╗
                 ║ CONTEXT BUILDER║                          ← assembly
                 ║  (runtime,     ║                            order +
                 ║   deterministik║                            positioning;
                 ║   non-generative)║                          bukan LLM
                 ╚═══════════╦═════╝
                             ▼
                      Qwen 7B
                   Sovereign Brain
                             ▼
                         response
```

Pembacaan penting: Identity/Behavior/State adalah **representasi**,
Memory/Relationship/Context adalah **store**, Context Builder adalah
**fungsi assembly deterministik**. Hanya Qwen yang generatif.

## 2. Definisi layer

### 2.1 IDENTITY
- **Fungsi**: referent diri ("siapa aku") + kebijakan challenge-response
  ("apa yang kulakukan saat label-ku diserang/diganti").
- **Evidence**: prompt-saja gagal gate identitas (31%); state minimal
  terbaca tapi tak grounding (IGE-B borderline); referent telanjang
  kolaps (IGE-C). Grounding sejati butuh weights [C.AI/BIG5 evidence].
- **Kandidat lokasi**: (a) tetap di examples+state (baseline riset);
  (b) demonstrated-resistance examples; (c) LoRA adapter (jangka
  panjang). **Hypothesis**: identity = kombinasi referen-state +
  demonstrated behavior; bukan deskripsi pasif.

### 2.2 BEHAVIOR
- **Fungsi**: effort matching, prosody, anti-service reflex, stance.
- **Evidence**: examples = permukaan kontrol terkuat (ablation:
  D_FEWSHOT 0% anti-service vs rules 17–30%); relapse kontekstual
  tetap terjadi tanpa enforcement posisi.
- **Kandidat lokasi**: few-shot examples (terbukti) + PHI-analog
  (enforcement ringkas di akhir konteks — lihat §3).

### 2.3 STATE
- **Fungsi**: situasi aktif, mode (social/technical), affect.
- **Evidence**: affect engine v1 sudah bekerja di jalur ini; ST memakai
  author's-note@depth utk refresh berkala; IGE menunjukkan state block
  memengaruhi framing walau lemah.
- **Kandidat lokasi**: runtime injection @depth (refresh), terpisah
  dari blok persona statis.

### 2.4 MEMORY
- **Fungsi**: fakta lintas sesi, event ringkas. Saat ini UTA TIDAK
  punya apa pun di layer ini (window only).
- **Kandidat lokasi**: store ringkasan + keyword/semantic retrieval,
  diinject kondisional (pola World Info / companion apps).

### 2.5 RELATIONSHIP
- **Fungsi**: bagaimana UTA memandang user (fakta user, sejarah
  interaksi, running jokes). Saat ini tidak ada.
- **Kandidat lokasi**: profil terstruktur selalu-diinject (pola
  Kindroid Key Memories / C.AI user persona).

### 2.6 CONTEXT (window)
- **Fungsi**: percakapan live. Sudah ada. Masalahnya hanya budget &
  komposisi (lihat §3).

### 2.7 MODEL PRIORS
- **Fungsi**: perilaku dasar dari RLHF Qwen — sumber assistant-prior.
- **Evidence**: hanya dapat digeser via training (PSM; kimjammer).
- **Kandidat jangka panjang**: LoRA referent-grounding pada Qwen —
  satu-satunya jalur yang menyelesaikan identity di bobot. Belum
  direkomendasikan dieksekusi; cukup sebagai feasibility study.

## 3. CONTEXT BUILDER — positioning strategy (runtime, deterministik)

Temuan inti riset: POSISI dalam konteks adalah kontrol tersendiri
[U-shaped bias, Liu et al.; PHI resmi SillyTavern]. Proposal assembly
order (hipotesis untuk diuji, bukan keputusan):

```
[system ringkas]            ← control contract pendek
[identity referent]         ← ramping
[behavioral examples]       ← D_FEWSHOT-class, termasuk challenge demos
[retrieved memory]          ← kondisional
[relationship profile]      ← selalu, tapi ramping
[state snapshot @depth-N]   ← refresh berkala, dekat akhir
[conversation history]
[turn-final enforcement]    ← PHI-analog: 1-2 kalimat, TERAKHIR
[user turn]
→ Qwen (satu generative call)
```

Aturan: tiap blok punya SATU fungsi; total persona-text budget jauh
di bawah Persona Plane v1 (~800 tok); tidak ada duplikasi aturan.

## 4. Yang secara EKSPLISIT ditolak proposal ini

- Multi-stage cognitive pipeline (JSON draft finalizer) — dibuktikan
  regresi parah oleh ADR-001.
- Persona constitution lebih panjang — satiasi terukur sendiri.
- Hard-rules tambahan untuk menambal failure per-kasus — jalur jenuh.
- Cloud sebagai pembentuk persona — melanggar Single-Brain.
- Menyimpan "persona" di database eksternal untuk kemudian di-paste
  utuh — hanya menggeser satiasi, bukan menyelesaikan.

## 5. Roadmap eksperimen berikutnya (urutan prioritas)

| # | Eksperimen | Menguji | Prasyarat |
|---|---|---|---|
| R1 | **PHI-analog positioning**: ulangi DGA battery dengan enforcement ringkas di akhir konteks | apakah recency-position menekan relapse ujung-percakapan | none |
| R2 | **State + challenge-examples**: D_FEWSHOT + demonstrated resistance (denial CS, deflection) + state block | apakah kombinasi referen+demonstrasi > masing-masing (IGE-2) | none |
| R3 | **Relationship profile prototype**: profil user ramping selalu-diinject vs tidak | dampak relationship layer thd konsistensi & anti-interogasi | R1 |
| R4 | **Memory retrieval prototype**: keyword-triggered fact injection ala World Info | dampak continuity tanpa merusak style | R1 |
| R5 | **LoRA feasibility study** (paper-level): data needs, hardware (RTX 4060 8GB cukup utk QLoRA 7B), expected trade-off | kelayakan jalur weights | R1–R4 hasil |

Setiap eksperimen: pre-registered gates, multi-run, artifact persisten
di experiments/, production untouched.

## 6. Kriteria sukses V2 (untuk masa depan, bukan sekarang)

Suatu saat Persona System V2 layak masuk produksi hanya jika:
1. Identity gate PASS (<10% fail multi-run) — prasyarat absolut.
2. Anti-service ≥ D_FEWSHOT baseline pada stimulus OOD.
3. Tidak ada relapse ujung-percakapan pada battery Part III.
4. Overhead token total ≤ Persona Plane v1.
5. Semua melalui SATU generative call Qwen per turn.
