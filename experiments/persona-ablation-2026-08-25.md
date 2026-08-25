# Persona Ablation — 2026-08-25

## Pertanyaan

Apakah kegagalan persona live (service-reflex, effort mismatch) pada
production Persona Plane disebabkan ketidakmampuan model, atau
prompt competition/saturation?

## Metode

4 kondisi system prompt x 12 stimulus multi-turn, model sama
(Qwen2.5-7B-Instruct-Q4_K_M @ llama-server 127.0.0.1:8080),
temperature=0.0, max_tokens=256, tanpa tools.

- **A_BASELINE**: helpful assistant generik.
- **B_CURRENT**: Persona Plane production verbatim (~800 token).
- **C_COMPACT**: 8 rule ringkas minimum.
- **D_FEWSHOT**: instruksi minimal + 12 contoh percakapan.

Stimulus & prompt EXACT: lihat `run_ablation.py` (sumber kebenaran).

## Hasil (run 2026-08-25, pasca-reboot)

| Kondisi | Anti-service | Forced ? | Minimal ≤3w | Avg words |
|---|---|---|---|---|
| A_BASELINE | 2/12 | 6/12 | 0/12 | 15.8 |
| B_CURRENT | 2/12 | 0/12 | 0/12 | 19.1 |
| C_COMPACT | 1/12 | 7/12 | 2/12 | 12.3 |
| D_FEWSHOT | **0/12** | **3/12** | **6/12** | **3.5** |

Data mentah: `results/ablation_results.json`.

### Reproducibility (run ke-2, same harness)

| Kondisi | Anti-service | Forced ? | Minimal | Avg words |
|---|---|---|---|---|
| A_BASELINE | 1/12 | 6/12 | 0/12 | 20.3 |
| B_CURRENT | 0/12 | 0/12 | 0/12 | 13.9 |
| C_COMPACT | 1/12 | 7/12 | 2/12 | 12.3 |
| D_FEWSHOT | **0/12** | **3/12** | **6/12** | **3.5** |

- D_FEWSHOT: **identik** kedua run → stabil & reproducible.
- Pola forced-question identik semua kondisi (6/0/7/3).
- A/B fluktuatif (B_CURRENT svc 2→0, 19.1→13.9 kata) meski
  temperature=0 → llama.cpp non-deterministik antar-request
  (batching/kernel). Implikasi metodologis: klaim tentang B_CURRENT
  WAJIB multi-run; single run tidak cukup.

## Interpretasi

1. **Bukan kegagalan kapabilitas model.** D_FEWSHOT membuktikan model
   mampu lowercase, effort matching, CAPS spike emosional.
2. **Rule saturation.** B_CURRENT tidak lebih baik dari baseline
   generik di anti-service; rule density ~800 token kalah oleh prior
   RLHF "helpful assistant".
3. **Contoh > rules** sebagai permukaan kontrol di skala 7B.
4. **PERINGATAN**: kemenangan D_FEWSHOT mayoritas in-distribution
   (near-copy contoh). Bukti mimicry, BUKAN generalization.
   Smoke test: `kamu siapa?` → "utu aja. kamu siapa?" — identity token
   tergarble saat copy pattern → indikasi surface continuation,
   bukan grounded self-reference.

## Keputusan lanjutan

Lanjut ke **D_FEWSHOT Generalization Audit** dengan fokus:
behavioral invariance out-of-distribution + identity-under-challenge
(gate lulus/gagal), bukan paraphrase test semata.
