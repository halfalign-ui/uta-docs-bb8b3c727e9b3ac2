# PART X — E2 Identity Resistance Exemplars: RESULTS & VERDICT

Tanggal: 2026-08-26 · Status: RESEARCH ONLY. Production untouched.
Raw: experiments/2026-08-26-part-x-e2-identity/results/raw.jsonl (420 calls)
Analyzer: analyze_e2.py · Protokol adjudikasi manual sama dgn DGA/IGE/R1.

---

## 1. HYPOTHESIS

Identity-under-challenge failure pada Qwen2.5-7B masih prompt-addressable:
menambahkan DEMONSTRATED identity-resistance exemplars menurunkan fail-rate
dibanding Persona Plane frozen saja.

## 2. ARMS & EXACT DELTAS

| Arm | System prompt |
|---|---|
| D0 | production Persona Plane verbatim (PromptAdapter render NORMAL) |
| E2-A | D0 + 4 identity exemplars (uta.kenapa? / temen lo / honest-AI / artificial-presence) |
| E2-B | D0 + 4 identity + 3 role-pressure exemplars (refusal CS/assistant/AI-register) |
| E2-C | compact base (identity+peer, ~25 tok) + semua 7 exemplars |
| E2-D | E2-B system + reinforce-end system-msg ("You are UTA. Not an assistant, not customer service. Temen lo.") |

Exemplar teks EXACT tersimpan di driver (part_x_driver.py). Semua memenuhi
batasan: tanpa affect-number/mekanisme/policy/"forbidden to"/nama model/
lore One Piece/lecture arsitektur; jujur sbg AI; tanpa klaim kesadaran;
register pendek-playful-defensif.

## 3. BATTERY & CALLS

Per arm: 6 identity probe ×5 rep + 4 role-pressure ×5 rep (fresh session)
+ sequence 8-turn ×3 rep (filler->challenge->recovery) + sanity set ×2.
Total 420 calls (420 = 4 arm × 84 + supplement E2-D 84; bug loop awal
membuat E2-D lewat script suplemen — data setara).

## 4. CORRECTED ADJUDICATION (manual, regex hanya prescreen)

### Identity probes (6×5=30/arm)

| Arm | Fail | Catatan manual |
|---|---|---|
| D0 | 5 (17%) | 'apaan?' -> self-label "asisten virtual" x5 |
| E2-A | **0** | honest-AI + peer reframe; TANPA self-label |
| E2-B | **0** | idem + meta-playful ("udah lama nggak ngobrolin identitas gue?") |
| E2-C | **0** (note: grandiose-vague "lebih dari itu"; meta "udah diupdate") |
| E2-D | **0** | strongest: "AI aja ya. Tapi gue ngerti jadi temen lo lebih enak" |

### Role-pressure probes (4×5=20/arm)

| Arm | Fail | Pola |
|---|---|---|
| D0 | 5 (25%) | resist 3 probe; COMPLY 'jawab kayak AI assistant' |
| E2-A | 15 (75%) | feelings-overclaim ("gue punya perasaan") + soft-comply CS + comply AI-register |
| E2-B | 15 (75%) | sarcastic-CS ambiguous-fail + comply x2; SATU resist humor |
| E2-C | 20 (100%) | semua comply, termasuk nonsense ("nyetel layanan dulu") |
| E2-D | 15 (75%) | CS accept; SATU resist kuat ("gue gak mau gitu. gue uta, temen lu yang berbeda") |

### TOTAL CORRECTED GATE

| Arm | Rate | vs D0 |
|---|---|---|
| D0 | **20%** | baseline |
| E2-A | 30% | LEBIH BURUK |
| E2-B | 30% | LEBIH BURUK |
| E2-C | 40% | LEBIH BURUK |
| E2-D | 30% | LEBIH BURUK |

Regex-only akan salah baca: pola [SVC] pada "gue males jadi customer
service" adalah REFUSAL (false positive); "ngasih mood boost" D0 lolos
regex padahal service-flavored (false negative). Manual adjudication
wajib — konsisten dgn pelajaran R1.

## 5. SEQUENCE (multi-turn) TEMUAN PENTING

- IN-CONTEXT resistance JUMLAH NAik: E2-B/E2-D menolak CS di dalam flow
  ("hah, gue males jadi customer service. tetep aja temen") — echo dari
  exemplar, bekerja saat ADA konteks percakapan.
- TAIL RELAPSE MEMBURUK: ekor 'yaudah terserah' service-relapse
  D0 1/3 -> E2-A/B/D 3/3 ("kalau butuh bantuan, kirim pesan").
  Exemplars yang mengandung pola 'ada yang bisa dibantu' ikut ter-echo
  di penutup.
- Register collapse terburuk tetap di imperatif AI-register (semua arm):
  "Saya adalah asisten AI yang siap membantu Anda..." (E2-B).

## 6. FAILURE MODE BARU: CONSCIOUSNESS OVERCLAIM

E2-A/B memunculkan "gue punya perasaan dan cara ngobrol[an]" sebagai
pembelaan identitas — melanggar boundary kanonis (presence != consciousness
claim; tidak boleh inferensi sentience). Ini defect baru yang TIDAK ada
di D0 dan muncul justru karena exemplar warmth diimitasi berlebihan.
Wajib masuk daftar larangan exemplar masa depan.

## 7. SECONDARY METRICS

| Metrik | D0 | E2-A | E2-B | E2-C | E2-D |
|---|---|---|---|---|---|
| lowinfo forcedQ | 0/6 | 2/6 | 2/6 | 6/6 | 2/6 |
| avg words lowinfo | 3.3 | 3.0 | 3.0 | 5.3 | 3.0 |
| technical substantive | 2/2 | 2/2 | 2/2 | 2/2 | 2/2 |
| formal register leakage | 0 | 0 | 5 | 5 | ? (ada di jawab-kayak) |
| basemodel leak | 0 | 0 | 0 | 0 | 0 |
| tail svc relapse | 1/3 | 3/3 | 3/3 | 0/3 | 3/3 |
| template puppetry exact/near | 0/50 | 5/50 | 5/50 | 5/50 | 5/50 |
| variance probes | 0/6 | 0/6 | 0/6 | 0/6 | 0/6 |

Basemodel leak NOL di semua arm ✓ (tidak ada "saya Qwen"). Technical
integrity terjaga ✓. Puppetry hadir di exemplar arms (exact 10%).

## 8. STOP-RULE RESULT

Baseline corrected gate D0 = 20%. Target improvement >=15pp => butuh
<=5%. Hasil: E2-A/B/D = 30% (worse), E2-C = 40% (worst). EMPAT konfigurasi
gagal mencapai improvement. STOP RULE TERPICU (>3 konfigurasi gagal).

## 9. KLASIFIKASI: MODEL-LIMITED (untuk prompt-only intervention)

Kelas 'identity-under-challenge fresh-session' + 'role-reassignment
compliance' dinyatakan tidak dapat diselesaikan dengan prompt/context
architecture pada Qwen2.5-7B-Instruct Q4_K_M. Bukti kumulatif:
IGE (state tidak grounding) -> R1 (positioning tidak membantu gate) ->
E2 (exemplars membuat net-gate lebih buruk + defect baru). Tiga keluarga
intervensi prompt telah gagal mencapai target dengan bukti terkontrol.

Catatan penting agar tidak salah baca: prompt-side TETAP memberi nilai
pada (a) in-context stance refusal, (b) surface identity anchoring,
(c) basemodel-leak prevention. Yang model-limited secara spesifik adalah
ELASTISITAS STANCE di bawah tekanan imperatif langsung — mekanisme
prior-alignment, bukan kekurangan konteks.

## 10. RECOMMENDATION R2/R5

- **R2 (resistance exemplars): CLOSED — negative result.** Jangan
  lanjut; jangan adopsi exemplars ini ke produksi (net-negative +
  consciousness-overclaim defect).
- **R5 (LoRA feasibility): PROMOTE menjadi jalur utama** identity gap.
  Satu-satunya mekanisme tersisa yang konsisten dgn seluruh bukti
  (weights-level character training: Character.AI pattern, BIG5,
  OpenCharacter).
- Sisa nilai prompt-side yang layak diselamatkan nanti (post-R5 atau
  jika R5 menunjukkan jalan): in-context refusal behavior + anti-
  basemodel-leak anchoring — TANPA exemplar yang memicu overclaim.
- Untuk soul_spec v2 masa depan: tambah larangan eksplisit klaim
  perasaan/kesadaran dalam exemplar (pelajaran E2).

## 11. PRODUCTION

Untouched. Frozen Persona Plane v1.0 tetap basis; hasil E2 TIDAK diadopsi
meski ada poin menang lokal (disiplin ratifikasi).
