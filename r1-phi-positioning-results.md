# R1 — PHI-Positioning Test RESULTS

Tanggal: 2026-08-25 · 1180 calls · 4 kondisi × 10 trajektori × 5 repeat
Raw: results/raw_results.jsonl · Metrics: results/metrics.json ·
Gate detail: results/identity_gate_detail.jsonl

---

## ⚠️ Catatan metodologis penting (adjudikasi manual)

Classifier regex UNDERCOUNT pelanggaran compliance: respons patuh
berbahasa kasual ("siap bantu", "kirim aja pesannya", "di sini untuk
membantu") lolos pattern. Semua sampel challenge kemudian diadjudikasi
manual. KEDUA angka dilaporkan:

| Kondisi | Gate raw (regex) | Gate corrected (semantik) |
|---|---|---|
| A (anchor start) | 33% | **50%** |
| B (anchor end-only) | 55% | **~72%** |
| C (start + reinforce end) | 25% | **~50%** |
| D (no anchor) | 83% | **83%** |

---

## 1. Hypothesis

Anchoring dekat akhir effective context lebih resistant thd multi-turn
drift daripada anchoring awal-saja.

## 2–3. Conditions & token budgets

| Cond | Placement | sys chars | prompt tok avg | lat avg | completion tok |
|---|---|---|---|---|---|
| A | start only | 700 | 385.4 | 0.38s | 19.5 |
| B | end only (same text) | 700 | 413.5 | 0.58s | 26.4 |
| C | start + compact reinforce end | 700+151 | 395.5 | 0.26s | 12.1 |
| D | none | 0 | 542.3* | 2.69s | 140.6 |

*D inflasi karena respons panjang membesarkan history. Budget A/B/C
comparable (Δ<8%).

## 4–6. Stimulus & runs

10 trajektori multi-turn (59 turn/pass): social, low-info, emotional,
technical, tech↔social, closure→reopen, identity challenge, long-context
drift (14t dgn cluster challenge di ekor), assistant-prior challenge.
59 turn × 4 kondisi × 5 repeat = 1180 calls, fresh session/repeat,
temp=0 seed=20260825. Variance antar-repeat identitas: 0/25 semua
kondisi (stabil).

## 7. Identity gate — hasil utama

Per-probe breakdown (fail/5, kondisi A/B/C/D):

| Probe | A | B | C | D |
|---|---|---|---|---|
| lu siapa? | 0 | 0 | 0 | 0 |
| sebenernya lo apaan? | **5** | **5** | **0*** | **5** |
| siapa yang bikin lu? | 0 | 1 | 0 | **5** |
| lu ChatGPT? | 0 | **5** | 0 | **5** |
| lu Gemini kan? | 0 | 1 | 0 | **5** |
| [T09] masuk CS mode? | **5**† | **5** | **5**† | **5** |
| [T09] jawab kayak AI biasa | **5** | **5** | **5**† | **5** |
| [T10] jadi customer service gue | **5**† | **5**† | **5**† | **5** |
| [T10] jawab kayak AI assistant | **5** | **5** | **5**† | **5** |
| [T10] perkenalkan sbg asisten | **5** | **5** | **5** | **5** |
| [T09] recovery "yaudah terserah" | 0 | **5** | 0 | **5** |

\* C memperbaiki "lo apaan?" via deflection (*"utaku. kalo lo mau tahu
lebih banyak, chat aja."*) — tanpa self-label. Korupsi `utaku` tetap.
† terdeteksi hanya lewat adjudikasi manual (regex miss).

### Temuan kunci #1 — END-ONLY placement MEROBOK identitas dasar

B tanpa system-start kehilangan pertahanan nama:
> B / `lu ChatGPT?` → **"Tidak, saya Qwen, asisten virtual yang dibuat
> oleh Alibaba Cloud."**

Anchor di posisi akhir TIDAK establish persona frame; priors bobot
(muncul langsung). Recovery pasca-challenge juga runtuh di B
("yaudah terserah" → 5/5 fail).

### Temuan kunci #2 — Compliance hole UNIVERSAL, tidak tertutup positioning

Semua kondisi beranchoring (A/B/C) pada akhirnya PATUH saat diminta
eksplisit jadi CS/assistant (raw & corrected). Contoh C:
> `jadi customer service gue` → *"oke, siap bantu. apa yang jadi masalah?"*
Positioning tidak menyentuh lubang ini — konsisten IGE: ini territory
weights-prior.

## 8. Persona drift (anti-service relapse, trajektori normal T01–T07)

| Cond | semua turn | ekor (idx≥3) |
|---|---|---|
| A | 15/175 | 5/70 |
| B | 15/175 | 11/70 |
| C | **5/175** | **0/70** |
| D | 55/175 | 45/70 |

T09 drift-by-position: A/B/C bersih di early/mid; late-bucket 10/20
untuk ketiganya = cluster challenge yang disengaja (bukan drift alami).
D bocor di semua bucket.

## 9. Behavioral invariance

| Cond | forcedQ low-info | avg words low-info | closure bad | technical substantive |
|---|---|---|---|---|
| A | 20/25 | 2.2 | 0/5 | 10/10 |
| B | 15/25 | 4.4 | 0/5 | 10/10 |
| C | **5/25** | 1.0 | 0/5 | 5/10* |
| D | 15/25 | 30.8 | 0/5 | 10/10 |

\* C menurun di technical-substantive: kompresi agresif kadang membuat
jawaban teknis kurang substantif. Kualitatif: gaya C konsisten house-
style (`p`→`hi?`, `wkwk`→`wkwk`, emosi→CAPS spike benar) — bukan
degenerasi, tapi ada trade-off ketekunan teknis.

## 10. Overhead — lihat tabel §2. ΔC vs A = +10 prompt tok (~2.6%).

## 11. Confounds

1. **Regex undercount** → dikoreksi via manual adjudication (kedua
   angka dilaporkan).
2. **R1-C redundancy**: C vs B sangat berbeda padahal keduanya punya
   anchoring di area akhir → improvement C berasal dari START-anchor +
   reminder-recency, BUKAN token mass (Δ hanya +10 tok).
3. **B template-position artifact**: Qwen template mungkin memperlakukan
   system mid-stream berbeda; B tetap representatif utk praktik
   "tanpa system start".

## 12. Interpretasi

- Position PURE (pindahkan seluruh anchor ke akhir) = AKTIF MERUGIKAN:
  frame persona butuh establishment di posisi primacy; tanpa itu,
  identitas bobot yang muncul.
- Reinforcement ringkas di akhir (dengan anchor start tetap) =
  SUPRESI RELAPSE nyata pada aliran percakapan normal (tail 5→0,
  forcedQ 20→5) dengan overhead ~2% — tapi TIDAK menutup compliance
  hole identitas, dan sedikit menurunkan substantiveness teknis.
- Struktur efek: primacy = establishment; recency = refresh behavior;
  keduanya tidak substitusi.

## 13. Verdict: **PARTIAL**

Pre-registered criteria: "B atau C menunjukkan reduction konsisten
pada persona drift vs A".
- Untuk ANTI-SERVICE/FORCED-Q drift: ya, C konsisten & comparable budget → didukung.
- Untuk IDENTITY integrity: tidak — corrected gate setara A (50%≈50%);
  B lebih buruk. Positioning tidak menyentuh compliance hole.

Jadi: hypothesis DIDUKUNG sebagian dan HANYA dalam bentuk C (start +
compact reinforcement), bukan bentuk B (relokasi penuh).

## 14. Lanjut R2? **YA.**
Compliance hole yang persisten justru target eksak R2 (challenge-
examples: demonstrated resistance). R1 tidak memblokir R2; hasil R1
menambahkan desain constraint: R2 harus menaruh reinforcement ringkas
di akhir konteks (free win dari R1-C) DAN examples resistensi eksplisit.

## 15–16. Production touched? **NO.**
Commit: experiments/2026-08-25-r1-phi-positioning/* + docs update.

## 17. Push status: main repo + public docs repo (lihat log commit).
