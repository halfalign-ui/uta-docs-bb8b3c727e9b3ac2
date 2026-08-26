# PART X-BRANCH — Ornith-1.5-35B-A3B Screening Report

Tanggal: 2026-08-26 · Status: SCREENING TERMINATED di PHASE 1 (feasibility).
Production untouched. R5 tetap frozen pada tag `r5-frozen-step2`.
27B tetap REFERENCE ONLY — tidak digantikan, tidak diadopsi.

---

## 0. R5 FREEZE CONFIRMATION

Tag git: `r5-frozen-step2` @ cc3f17d (pushed).
D-Small dataset CONDITIONAL; curation/training L1 ditunda tanpa batas
waktu sampai otorisasi owner.

## 1. MODEL VERIFICATION (Phase 0)

| Item | Hasil | Status |
|---|---|---|
| Model identity | ornith-ai/Ornith-1.5-35B-A3B (HF org resmi) | VERIFIED [F] |
| Architecture | Qwen3.5 MoE (qwen3_5_moe): 40 layer + native MTP, 256 experts / 8 active, hidden 2048, vocab 248k | VERIFIED [F] |
| Parameters | 35B total / ~3B active per token | VERIFIED [F] |
| Weights | HF safetensors + FP8 resmi; GGUF komunitas (ornith-ai resmi, bartowski, SC117-MTP-APEX) | VERIFIED [F] |
| Quantization | Q2_K s/d Q8 + UD variants tersedia | VERIFIED [F] |
| License | MIT (base); MTP head Apache-2.0 via Qwen | VERIFIED [F] |
| Context | 262,144 tokens; multimodal (mmproj) | VERIFIED [F] |
| Reasoning model | default <think> block ON | VERIFIED [F] |
| llama.cpp support | butuh rilis >= b10472; build kita (9e40df6, Aug-14) kemungkinan perlu rebuild | PARTIAL [I] |

Kandidat BUKAN hoax — keluarga Ornith nyata, benchmarks agentic kuat
(Terminal-Bench 67.8, SWE-Bench Verified ~79) pada hardware kelas server.

## 2. LOCAL RUNTIME FEASIBILITY (Phase 1) — GAGAL DI ARITMETIKA

Hardware UTA: RTX 4060 8GB · i5-10400 · **15GB RAM** · llama.cpp.

| Kebutuhan | Nilai | vs Kita |
|---|---|---|
| GGUF Q4_K_M file | ±20GB | > total RAM 15GB |
| RAM minimum (CanIRun, Ornith-1.0 same-arch) | **19.6GB min / 32.6GB rec** | **di bawah floor fisik** |
| Serving resmi | vLLM/SGLang, 2×80GB class | jauh di luar kelas |
| Coexistence dgn 7B always-on | 7B pegang 6.2GB VRAM + cache | memperparah |

Analisis:
1. Semua quant yang menjaga kualitas (Q4+) melebihi RAM fisik.
2. Argumen "MoE hanya 3B aktif, bisa mmap-stream dari disk": sah secara
   teori, tapi 256 experts dirouting acak -> NVMe random-read per token;
   ditambah bukti empiris PART V (degradasi refault bahkan pada model
   yang MUAT di RAM) -> operasi stabil tidak realistis.
3. Coexistence dengan 7B presence-brain menggandakan tekanan cache.
4. Build llama.cpp kita kemungkinan perlu rebuild untuk varian ini
   (kartu bartowski menyatakan butuh >= b10472).

Keputusan: TIDAK dilakukan download/load attempt — aritmetika RAM floor
sudah menutup kasus sebelum biaya ±20GB download dikeluarkan. Ini sesuai
instruksi: jangan kejar konfigurasi yang membuat sistem unusable.

## 3–5. CAPABILITY SCREEN: TIDAK DAPAT DIJALANKAN

T1–T4, perbandingan 7B/A3B/27B, tok/s, tool-calls: **N/A** — tidak ada
konfigurasi runnable. Tidak ada angka palsu dilaporkan.

Untuk konteks (bukan evidence internal): benchmark kartu model melaporkan
kekuatan agentic/coding pada kelas hardware server [F-vendor]. Angka itu
TIDAK dapat ditransfer ke skenario lokal kita.

## 6. PERBANDINGAN

| Aspek | Qwen2.5-7B (control) | Ornith-1.5-35B-A3B | Qwen3.8-27B (ref) |
|---|---|---|---|
| Muat di hardware UTA | YA (production) | **TIDAK** | ya, mode CPU-only lambat (PART T) |
| Generation speed lokal | ~40-50 t/s GPU | N/A (tidak runnable) | 1.4 t/s coexistence |
| Identity/UTA behavior terukur | ada (DGA/IGE/E2/R1) | tidak ada | tidak dieksplorasi utk persona |
| Kekuatan reasoning (paper) | dasar | kuat [F-vendor] | kuat [F-vendor] |
| Biaya operasi | nol tambahan | butuh upgrade RAM>=24-32GB | nol tambahan |

## 7. FAILURE MODES PENCATATAN

FM-A1: RAM floor terlewati (19.6GB min vs 15GB fisik).
FM-A2: tidak ada jalur quant yang muat tanpa degradasi kualitas ekstrem.
FM-A3: routing acak 256 experts + mmap streaming = thrash pada box ini.
FM-A4: coexistence dengan presence-brain menggandakan tekanan.
FM-A5: thinking-mode default + latensi CPU = UX tak layak walau berhasil load.
FM-A6: build llama.cpp lokal kemungkinan pra-dukungan penuh varian ini.

## 8. CONFIDENCE & LIMITATIONS

- Keputusan berbasis ARITMETIKA kapasitas + spesifikasi resmi, bukan
  benchmark empiris lokal (mustahil dijalankan) — confidence HIGH untuk
  infeasibility, dan UNKNOWN utk capability advantage sebenarnya.
- Hypothesis awal bersumber video eksternal: klaim kekuatan model TIDAK
  dibantah — yang dibuktikan adalah ketidakcocokan hardware.
- Vendor benchmark = [F-vendor]; belum independen.

## 9. DECISION GATE

Gate A/B/C/D TIDAK TERCAPAI — Phase 1 gagal struktural.
Klasifikasi operasional: **NOT FEASIBLE ON CURRENT HARDWARE.**
Sesuai aturan branch: jika C/D atau tidak-runnable -> TERMINATE.

## 10. RECOMMENDATION

**DROP branch ini untuk hardware saat ini.** Bukan karena model jelek —
benchmark-nya nyata — tapi karena parameter-parameter itu tidak bisa
hidup di mesin kita.

REOPEN CONDITIONS (salah satu):
1. Upgrade RAM >= 32GB (+ idealnya GPU 24GB) -> ulangi Phase 1.
2. Rilis GGUF distill/kompres <10GB berkualitas utk keluarga ini.
3. Ornith di-host sebagai CLOUD external cognition -> eksperimen BERBEDA
   (masuk slot External Cognitive Resources ADR-001, bukan local brain);
   butuh proposal terpisah.

Anchor question terjawab jujur:
"A3B memberi lebih banyak otak daripada 7B?" — Di hardware kita: TIDAK,
karena otaknya tidak bisa dimuat. Yang ada saat ini cuma lebih banyak
parameter di atas kertas (dan di atas anggaran RAM).
