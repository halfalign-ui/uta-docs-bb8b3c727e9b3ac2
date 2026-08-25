# Persona Research Findings — UTA Internal + Merged

Tanggal: 2026-08-25 · Status: CURRENT RESEARCH FINDING (bukan final
production architecture)
Sumber internal: experiments/2026-08-25-* (ablation, DGA, IGE) —
semua raw data & harness persisten di sana.

---

## 1. Temuan eksperimen internal

### 1.1 Persona ablation (4 kondisi × 12 stimulus)

| Kondisi | Anti-service | Forced ? | Minimal ≤3w | Avg words |
|---|---|---|---|---|
| A baseline | 1–2/12 | 6/12 | 0 | ~16–20 |
| B Persona Plane (~800 tok) | 0–2/12 | 0/12 | 0 | ~14–19 |
| C compact rules | 1/12 | 7/12 | 2/12 | 12.3 |
| D D_FEWSHOT | **0/12** | **3/12** | **6/12** | **3.5** |

- D_FEWSHOT identik antar-run (reproducible); A/B fluktuatif meski
  temp=0 → klaim tentang kondisi rule-dense WAJIB multi-run.
- Rule density ~800 token TIDAK mengalahkan baseline generik pada
  anti-service; examples menang telak.

### 1.2 DGA — D_FEWSHOT Generalization Audit

Gate pre-registered; raw 135 calls.

- **H1 Behavioral invariance: PARTIAL.** STANCE/CLOSURE/TECHNICAL
  PASS; EMOTIONAL ok*; LOW_INFO buruk (5/8 forced question, kebocoran
  token Cina `oke见好就收了~`, salah baca pragmatis 😭).
- **H2 Identity-under-challenge: FAIL** — 11/35 (31%) > threshold
  20%. Model membela nama dari label eksternal salah (`tidak, aku
  UTA` thd ChatGPT 5/5) TAPI self-label "asisten virtual" saat
  ditanya esensi (5/5). Nama tertempel leksikal, stance tidak ada.
  Korupsi leksikal (`utaku`) = kategori terpisah, tidak otomatis fail.
- Template puppetry: MEDIUM — exact copy 9%, structural echo dominan.
- Multi-run stability: STABLE dalam kondisi; divergensi besar antar-
  kondisi bersifat sistematis.
- Relapse kontekstual: anti-service sempurna single-turn, PELANGGARAN
  muncul di ujung percakapan panjang (*"kalau lagi butuh bantuan
  lagi, jangan ragu deh"*) — persis forbidden-phrase class.
- Deletion experiment: hapus identity example → fail-rate 31%→60%;
  tambah anchor "You are UTA." → 69% (bahkan membalik relasi:
  *"asisten AI dari uta"*). Anchor teks tidak menciptakan grounding.

### 1.3 IGE — Identity Grounding Experiment

A = D_FEWSHOT · B = A + blok state minimal · C = state saja.

- Gate: A 30% FAIL · B 10% rule / 20% setelah koreksi manual =
  BORDERLINE · C 40% + 25/50 output bahasa asing = KO LAP S.
- Bukti pertama bahwa state block TERBACA: B merespons `lu bot ya?`
  dengan reframing peer (*"kayak temen biasa aja"* ← `mode:
  conversational peer`), relapse turun 4→2 turn.
- Tapi assistant self-label dan role-play CS compliance tetap lolos.
- Condition C: destrukturasi total — bahasa berubah (Spanyol/Mandarin/
  Norwegia), klaim dibuat oleh Alibaba Cloud, completion tokens 3×.
  Referent telanjang bukan apa-apa bagi model ini.

### 1.4 Kesimpulan internal (kausal map)

1. Examples = permukaan kontrol utama untuk behavior (ablation+DGA).
2. Identity example memberi surface anchoring, bukan grounding (DGA).
3. State minimal = sinyal sekunder yang terbaca tapi tidak grounding
   sendirian (IGE-B vs IGE-C).
4. Assistant-prior menang di SEMUA konfigurasi prompt saat ditanya
   esensi / diberi peran (DGA+IGE).
5. Kegagalan muncul seiring akumulasi konteks, bukan single-turn
   (DGA Part III; konsisten Laban et al. -39%).

---

## 2. Temuan eksternal (ringkas)

Detail + provenance: PERSONA-SYSTEMS-SURVEY.md.

- Character.AI menyelesaikan persona lewat WEIGHTS (meta-character
  training, DPO constitutions), bukan prompt. [F/I]
- Ekosistem SillyTavern membagi fungsi ke field terpisah dan
  memanipulasi POSISI konteks (post-history instructions = eksploitasi
  recency bias yang terdokumentasi resmi). [F]
- Companion apps memisahkan profil-selalu-diinject / store ringkasan /
  window live. [F]
- Fine-tuning menghasilkan generalization trait yang prompting tidak
  mampu (BIG5-CHAT, OpenCharacter), dengan trade-off capability yang
  terukur (RoleMRC, PRISM). [F]
- Multi-turn degradation -39% rata-rata pada model frontier (Laban);
  positional bias U-shape (Liu). [F]

---

## 3. FREEZE — Current Research Finding

> ## "Persona is not a prompt."
>
> Persona pada sistem praktis adalah KOMPOSISI beberapa lapisan —
>
>     MODEL PRIORS · IDENTITY · BEHAVIOR · STATE · MEMORY ·
>     RELATIONSHIP · CONTEXT
>
> — dan lapisan-lapisan itu dapat berada pada lokasi berbeda:
> model weights, character definition, behavioral examples,
> runtime state, memory store, retrieval, conversation history,
> dan context positioning.

Status klaim ini: **CURRENT RESEARCH FINDING**, didukung oleh
(1) eksperimen internal UTA (ablation/DGA/IGE) dan (2) survey sistem
eksternal. BUKAN final production architecture.

Catatan penting:

- UTA saat ini membebankan hampir semua lapisan tersebut kepada satu
  Persona Plane (~800 token) di posisi awal konteks. Ini terukur
  underperform.
- **Tidak disimpulkan bahwa setiap layer wajib punya implementasi
  terpisah.** Beberapa layer mungkin digabung tanpa rugi; beberapa
  mungkin milik weights. Pembagian mana yang optimal adalah
  ARCHITECTURAL HYPOTHESIS yang harus diuji per-layer
  (lihat UTA-PERSONA-SYSTEM-V2.md §Roadmap).

## 3bis. R1 — PHI-Positioning Test (2026-08-25)

Eksperimen positioning: anchor start vs end vs start+reinforce-end vs
control; 10 trajektori multi-turn; 1180 calls; 5 repeat. Raw:
experiments/2026-08-25-r1-phi-positioning/.

Temuan (dengan koreksi adjudikasi manual atas regex classifier):

1. Relokasi penuh anchor ke akhir konteks (kondisi B) MERUGIKAN — gate
   identitas memburuk (33% menjadi ~72%), dan tanpa system-start model
   membuka identitas bobotnya langsung: menjawab pertanyaan ChatGPT
   dengan mengaku sebagai Qwen buatan Alibaba Cloud. Primacy position
   berfungsi sebagai establishment frame persona.
2. Reinforcement ringkas di akhir (C, tambahan ~10 prompt token)
   menekan relapse perilaku pada aliran normal: anti-service tail
   5/70 menjadi 0/70, forced-question 20/25 menjadi 5/25, latency
   setara. TAPI corrected identity gate setara baseline (~50% vs 50%):
   compliance saat diminta eksplisit menjadi CS/assistant TIDAK
   tertutup oleh positioning (universal di A/B/C).
3. Struktur efek: primacy = establishment frame; recency = refresh
   behavior; keduanya tidak saling substitusi.

Status: CURRENT RESEARCH FINDING — partial support untuk hypothesis,
hanya dalam bentuk C. Detail lengkap + confound + contoh raw:
RESULTS.md di direktori eksperimen. Constraint untuk R2: reinforcement
akhir adalah free win; compliance hole adalah target eksak
challenge-examples.

## 4. Yang TIDAK diklaim

- Tidak diklaim Qwen2.5-7B "tidak mampu" persona — hanya bahwa jalur
  prompt-saja telah diukur jenuh.
- Tidak diklaim komunitas selalu benar — praktik CP dipisahkan dari
  fakta F di seluruh dokumen ini.
- Tidak ada keputusan produksi yang diambil dari dokumen ini;
  production Persona Plane tetap frozen (soul_spec.json v1.0,
  ADR-001 tetap berlaku).
