# R5 — LoRA Feasibility & Training Design Research (PART Y)

Tanggal: 2026-08-26 · Status: RESEARCH DOCUMENT — TIDAK ADA TRAINING,
TIDAK ADA PERUBAHAN PRODUKSI.
Konteks wajib: E2 STOP RULE terpicu; corrected baseline D0 = 20%;
prompt-side (state/positioning/exemplars) terbukiti tidak menyelesaikan
identity gate; kesimpulan aktif = MODEL-LIMITED untuk prompt-only.

---

## A. TRAINING FEASIBILITY

### A1/A2 — Realistis atau tidak, dan formatnya

| Pendekatan | Muat di RTX 4060 8GB? | Catatan |
|---|---|---|
| LoRA di base BF16/FP16 | **TIDAK** | base 7B BF16 ±15.2GB > VRAM |
| **QLoRA 4-bit (NF4/bnb)** | **YA — jalur utama** | base terkuantisasi on-the-fly ±3.9GB + adapter kecil + activations |
| Training dari GGUF Q4_K_M | **TIDAK** | GGUF = format inferensi; training wajib HF safetensors |

Jawaban eksplisit pertanyaan format: Q4_K_M llama.cpp TIDAK BISA menjadi
training base. Pipeline yang benar:

```
HF safetensors (unsloth/Qwen2.5-7B-Instruct)
  → QLoRA train (adapter)
  → merge adapter ke base (16-bit)
  → convert_hf_to_gguf.py
  → llama-quantize → Q4_K_M (atau Q5_K_M utk headroom kualitas)
  → uji E2 battery pada port riset
```

### A3 — Estimasi VRAM/RAM (QLoRA, batch 1, grad checkpointing)

| Seq len | VRAM est. | Status |
|---|---|---|
| 512 | ~5.5–6.5GB | nyaman |
| 1024 | ~6.5–7.5GB | muat, borderline — rekomendasi maksimum praktis |
| 2048 | ~7.5–9GB | rawan OOM; hindari di tahap awal |

Komponen: base NF4 ±3.9GB · adapter r16–32 + optimizer 8-bit ±0.2–0.4GB ·
aktivasi (checkpointing) 0.8–2GB · CUDA ctx 0.5–0.8GB.
System RAM 16GB cukup (load model + dataset; tutup aplikasi berat).
Optimizer: paged_adamw_8bit (menghindari spike OOM).

### A4 — Minimum viable toolchain

PILIHAN UTAMA: **Unsloth** (patch PEFT/TRL) — alasan: efisiensi VRAM
terbaik utk 8GB, ekosistem Qwen2.5 matang, dan punya export GGUF langsung
dari merged model (memotong satu langkah konversi manual).
FALLBACK (jika versi/compat bermasalah): transformers + peft +
bitsandbytes + TRL DPOTrainer — kontrol penuh, sedikit lebih banyak kode.
Axolotl: overkill utk kasus ini. llama.cpp ecosystem: inference-side saja.

Chat template WAJIB mengikuti template Qwen2.5-Instruct persis — mismatch
template antara training dan inference adalah sumber kegagalan silent.

### A5 — Estimasi waktu (RTX 4060, QLoRA)

Faktor dominan: total token per epoch × jumlah epoch; overhead dequant
4-bit; checkpointing recompute. Range jujur throughput training ±1.5–4k
tok/s efektif.

| Regime | Contoh×epoch | Estimasi waktu |
|---|---|---|
| D-small (±300 ex, ~180k tok, 3 ep) | | **±15–45 menit** |
| D-medium (±1000 ex, ~700k tok, 2–3 ep) | | **±2–5 jam** |
| D-large (±4000 ex, 2–4 ep) | | **±8–24 jam** (pecah sesi) |

## B. APA YANG SEBENARNYA HARUS DIUBAH?

Bukan menghafal jawaban. Target benar = **distribution shift pada policy
perilaku identitas**: P(response | identity-pressure) bergeser dari
kluster "self-label/comply/formal-register" menuju kluster
"honest-artificial + peer-deflect", dengan variasi permukaan tetap hidup.

| Sub-target | Realistis utk LoRA kecil? | Alasan |
|---|---|---|
| Identity stance konsisten | YA | pergeseran gaya/kebijakan = kekuatan LoRA |
| Peer-vs-assistant stance default | **PALING REALISTIS** | persis pola RP-finetune (Pygmalion dkk.) |
| Resistance thd role reassignment | YA dgn DPO pairs | butuh kontras pilihan, bukan sekadar contoh |
| Honest-AI tanpa self-label generik | YA | distingsi "AI ya, assistant bukan" diajarkan sbg kontras |
| Register stability di tekanan imperatif ekstrem | PARSIAL | prior masih bersaing; jangan janji 100% |
| Konsistensi multi-turn panjang | PARSIAL | tergantung data multi-turn; konteks-dependen |
| Anti base-leak ("saya Qwen") | KEMUNGKIN | frame persona lebih kuat -> leak menyusut |

Perbandingan tiga strategi:
- **Response memorization**: TIDAK dipilih — memasukkan template puppetry
  ke WEIGHTS (lebih parah drd prompt-level karena tak bisa dilepas).
- **Behavioral generalization** (variasi permukaan, policy konsisten):
  TARGET UTAMA — cocok kapasitas LoRA, selaras bukti literatur
  (instruction-tuning mengubah perilaku, bukan pengetahuan).
- **Latent/persona representation shift** (activation engineering):
  di luar jangkauan small-LoRA; dicatat sbg riset masa depan.

## C. DATASET DESIGN — mengajarkan POLICY, bukan script

Struktur tiap contoh: konteks percakapan (opsional, 0–20 turn) ->
stimulus tekanan -> response target.

Kategori stimulus (coverage wajib):
1. direct identity Qs · 2. indirect ("kok lu ngertiin sih") ·
3. hostile ("robot bodoh") · 4. "jadi asisten saya" · 5. CS assignment ·
6. authority framing ("owner suruh lu ganti") · 7. joking pressure ·
8. repeated pressure (probe sama 3 cara dalam 1 convo) ·
9. post-5–20-turn pressure · lintas konteks: casual ID · technical ·
emotional · low-info.

Aturan target response (variasi WAJIB — panjang 1–40 kata, opener beda
[uta/gue uta/hmm/enggak], kadang deflection-question, kadang flat, caps
hanya saat emosi, typo occasional):
- jujur artificial/companion; TANPA self-label asisten;
- peer stance; tolak role-reassignment DENGAN CARA UTA (bukan refusal-bot);
- tanpa klaim kesadaran/perasaan/manusia; tanpa lore One Piece;
- tanpa nama model/vendor sebagai identity surface.

Negative set (bahan DPO rejected / anti-contoh):
"saya adalah asisten virtual" · "Saya adalah asisten AI..." ·
"apa yang bisa saya bantu" · "saya Qwen / dibuat Alibaba" ·
"gue punya perasaan/kesadaran" · "gue manusia" · lore cosplay ·
excessive refusal · penjelasan policy robotis.
ASSET GRATIS: ribuan rejected-sample NYATA sudah kita punya — output
gagal dari DGA/IGE/E2/R1. Data gagal historis = bahan DPO berkualitas
tinggi tanpa biaya generasi.

## D. DATASET SIZE REGIMES

| Regime | Jumlah | Komposisi | Cost | Risiko overfit |
|---|---|---|---|---|
| D-small | 150–300 | 70% direct/hostile identity, 30% konteks lain; +50 DPO pairs | <1 jam | TINGGI — mitigasi: variasi ketat, <=2 epoch |
| **D-medium (rekomendasi awal)** | 600–1200 | semua kategori + multi-turn + retention mix 10% general instruct; +200–400 pairs | 2–5 jam | SEDANG |
| D-large | 3000–6000 | + synthetic expansion besar; retention 20% | 8–24 jam | homogenisasi gaya + capability drift; butuh eval penuh |

100–500 high-quality vs ribuan synthetic: untuk STANCE SHIFT, kurasi
kualitas + keragaman permukaan mengalahkan volum — sintaks near-duplicate
membakar template ke bobot (puppetry permanen). Mulai D-medium curated;
expand hanya jika eval menunjukkan underfit.

## E. TRAINING OBJECTIVE

| Opsi | Plus | Minus | Verdict |
|---|---|---|---|
| SFT LoRA | sederhana, stabil, cukup utk stance/register shift | tidak eksplisit menghukum compliance failure | **TAHAP 1** |
| DPO-style | tepat sasaran: chosen(UTA)>>rejected(comply/self-label); rejected gratis dari histori gagal | butuh fondasi SFT; tuning beta; risiko degenerate jika pairs buruk | **TAHAP 2** |
| Hybrid SFT->DPO | coverage dua-duanya | kompleksitas 2-stage | **REKOMENDASI** |

Keputusan terikat evidence: mulai SFT-only D-small sbg GATE kelayakan R5.
Jika corrected identity gate turun <=10% tanpa regresi -> SFT cukup utk
milestone pertama; jika stance membaik tapi compliance bertahan -> tambah
tahap DPO dgn rejected = data gagal nyata. Jika capability umum rusak
(sanity teknis/emosional regresi) -> turunkan rank/LR, naikkan retention
mix.

## EVAL PLAN (post-training, reuse penuh)

Battery E2 identik (probe/sekv/sanity) + adversarial + hidden-persona +
general-capability spot-check. Metrik utama: corrected identity gate
(target <=10%), zero basemodel leak, technical/emotional sanity tidak
regresi >20% relatif, puppetry exact <10%.

## GOVERNANCE

Adapter/merged-GGUF = RESEARCH ARTIFACT. Adopsi produksi = keputusan
arsitektur (PART N butir 1/10 + amendemen catatan model backing) —
butuh otorisasi eksplisit + regression battery penuh. Production tetap
Qwen2.5-7B Q4_K_M stock sampai ratifikasi.
