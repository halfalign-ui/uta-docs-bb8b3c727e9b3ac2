# Persona Systems Survey — Eksternal

Tanggal: 2026-08-25 · Status: RESEARCH DOCUMENT (bukan spec produksi)
Provenance: setiap klaim berlabel [F] FACT / [I] INFERENCE / [CP]
COMMUNITY PRACTICE / [?] UNKNOWN. Sumber lengkap di
PERSONA-RESEARCH-REFERENCES.md.

Pertanyaan pemandu: bagaimana sistem AI praktis membuat persona tetap
konsisten padahal model dasarnya punya generic assistant prior?

---

## 1. Character.AI (proprietary — yang terdokumentasi publik)

**[F]** Dari kompilasi riset publik (blog resmi + analisis Nathan
Lambert/interconnects): arsitektur 4 lapis —

| Layer | Mekanisme |
|---|---|
| Foundation model | awalnya pre-train sendiri; pasca-2024: model third-party + post-training proprietary |
| **Character Training** | **DPO dengan personality constitution sintetis first-person ("I am ...")** — melatih KEMAMPUAN menjelma karakter apa pun dari deskripsi, bukan menjadi satu karakter |
| Prompt layer | definisi karakter (name/description/greeting/examples), tool "Prompt Poets" (YAML+Jinja); smart truncation memprioritaskan definisi karakter di atas history saat konteks penuh |
| User feedback | star rating → seleksi respons per-karakter; bukan perubahan bobot |

Konsep kunci: **meta-character training** — kemampuan menjelma ada di
WEIGHTS; deskripsi karakter hanya parameter runtime. [I] Ini jawaban
industri atas persis masalah UTA: assistant-prior dilawan di level
weights, bukan prompt.

Memory [F]: rolling window (~8k free tier) + definisi karakter selalu
di-inject + pinned memories/facts. Tidak ada memory persisten
sesungguhnya; keluhan "amnesia" adalah konsekuensi desain biaya.

**[?]** Detail internal (dataset, hyperparameter, pipeline persis)
tidak dipublikasikan.

## 2. SillyTavern / Character Card ecosystem (open source)

**[F]** Semua berikut dari dokumentasi resmi dan spesifikasi komunitas:

Character Card V2 fields dan fungsi aktualnya:

| Field | Mengontrol | Lapisan |
|---|---|---|
| `description` | identitas/fakta/knowledge | identity |
| `personality` | trait perilaku | behavior |
| `scenario` | situasi saat ini | state |
| `first_mes` | anchor gaya/panjang TERKUAT (model meniru opening lebih dari instruksi) | style |
| `mes_example` | voice via few-shot | behavior/style |
| `system_prompt` | aturan global (boleh menggantikan milik user) | rules |
| `post_history_instructions` | instruks SETELAH history — prioritas tertinggi | enforcement |
| lorebook/world info | fakta kondisional keyword-triggered (scan depth, token budget) | memory/retrieval |
| author's note @depth-N | injeksi berulang N kedalaman dari bawah | state/rules refresh |

Dua temuan spesifik yang relevan langsung ke UTA:

1. Spec V2 menyatakan eksplisit: *"system instructions written after
   the conversation history have a much stronger weight on current
   models' generations than instructions written before"*. Komunitas
   mengeksploitasi recency bias sebagai fitur (nama komunitas:
   Jailbreak/UJB). **[F]**
2. Docs ST: setelah history panjang terbentuk, efek main prompt
   menurun — *"the AI assumes the main prompt occurred in the distant
   past, and the message history updates it"*. **[F]** Validasi
   harfiah relapse ujung-percakapan UTA (DGA Part III).

## 3. Companion apps (Kindroid / Nomi / Replika)

**[F]** dari help center / docs resmi masing-masing:

| App | Persona/state | Memory |
|---|---|---|
| Kindroid | Backstory field besar | Key Memories di-inject TIAP turn; journal keyphrase recall; tiered cascaded memory |
| Nomi | Identity Core + backstory persisten | layered short/medium/long-term; auto-written notes; user-editable Shared Notes |
| Replika | persona muncul dari interaksi | memory terekstraksi otomatis + dashboard koreksi; upvote = reinforce |

**[I]** Pola umum: persona hidup dari (1) profil terstruktur selalu-
diinject, (2) store ringkasan yang diretrieve, (3) window live. Tiga
lapisan terpisah, bukan satu teks.

## 4. Local/small-model roleplay ecosystem

**[CP]** Praktik komunitas (r/SillyTavernAI, r/LocalLLaMA — evidence
praktik, bukan authoritative architecture):

- Example dialogue = kontrol perilaku terkuat (*"extremely
  underutilized and extremely powerful"*); 4–6 contoh variatif.
- First message menentukan gaya/panjang lebih dari instruksi mana pun.
- Trait list pendek > esai kepribadian; kartu ~1000-token diklaim
  "cuts the AI's memory in half" pada model konteks kecil (satiasi).
- Manual recap tiap ±15 turn untuk retensi.
- Format deskripsi: PList / Ali:Chat / W++ (legacy) / plain prose.
- Model RP fine-tuned (Pygmalion dkk.) dipilih karena stay-in-character.

**[CP]** Model roleplay lokal hampir tidak pernah instruct-generic
telanjang; biasanya fine-tune RP atau abliterated 8–13B.

## 5. Fine-tuning / LoRA untuk persona

**[F]** Paper peer-reviewed & preprint:

- **BIG5-CHAT** (ACL 2025): SFT/DPO menghasilkan profil kepribadian
  psikometris valid; prompting tidak. Korelasi antar-trait mendekati
  manusia hanya via training.
- **OpenCharacter** (2501.15427): 20K karakter sintetis + SFT →
  kualitas RP setara GPT-4o. Generalizable lintas karakter.
- **RoleMRC** (2502.11387): model RP fine-tuned skor LEBIH RENDAH
  dari instruct seukuran pada knowledge/instruction-following →
  trade-off nyata.
- **PRISM** (2603.18507): persona adapter meningkatkan alignment tapi
  merusak knowledge (MMLU 71.6→68.0); mitigasi: routing gate.
- **LoRA Learns Less, Forgets Less** (2405.09673): kapasitas LoRA <
  full FT; cukup untuk style/behavior, kurang untuk knowledge baru.
- **Anthropic PSM** (alignment blog, Feb 2026): perubahan perilaku
  saat fine-tuning dimediasi representasi persona internal — persona
  objek nyata di weights.
- **Multi-turn RL consistency** (OpenReview): fine-tuning konsistensi
  multi-turn tidak merusak instruction-following secara material.

**Jawaban**: fine-tuning memperbaiki GENERALIZATION trait (perilaku
konsisten tanpa example, tahan konteks panjang). Trade-off: data/GPU,
catastrophic forgetting, degradasi capability domain lain (dapat
dimitigasi adapter+router).

## 6. Multi-turn consistency & positional bias

**[F]** Paper:

- **Lost in the Middle** (Liu et al., TACL 2023): performa U-shape —
  info di TENGAH konteks paling mudah terlewat, termasuk di model
  long-context. Dasar ilmiah teknik PHI / AN@depth.
- **LLMs Get Lost in Multi-Turn Conversation** (Laban et al., MSR/
  Salesforce 2025): rata-rata **-39% performa multi-turn vs single-
  turn pada 15 model frontier**; mekanismenya kenaikan UNRELIABILITY
  (asumsi prematur, gagal recover) — bukan hilangnya aptitude.
- **When "A Helpful Assistant" Is Not Really Helpful** (2311.10054):
  162 persona × 2410 soal × 9 model — efek persona-via-prompt
  mayoritas nol/negatif dan tak prediktabel.
- **kimjammer finding** [CP]: aligned model menolak override
  prompt-level atas perilaku safety-trained — struktur sama dengan
  assistant-prior yang melawan persona.

## 7. Sintesis

**[I]** Tidak ada satu sistem serius yang menempatkan seluruh persona
sebagai satu blok teks statis di awal konteks. Persona praktis =
komposisi multi-lokasi: weights + character definition + examples +
runtime state + memory/retrieval + conversation history + positioning
strategy. Detail peta: PERSONA-SYSTEMS-MAP.md.
