# Persona Experiment Roadmap — Status

Terhubung: docs/persona/UTA-PERSONA-SYSTEM-V2.md §5 (roadmap R1–R5).
Update per eksperimen selesai. Production tetap frozen.

| ID | Eksperimen | Menguji | Status | Hasil ringkas |
|---|---|---|---|---|
| — | Persona ablation | rules vs few-shot vs compact | DONE 2026-08-25 | D_FEWSHOT menang; B_CURRENT saturasi |
| — | DGA | behavioral invariance OOD + identity gate | DONE 2026-08-25 | H1 PARTIAL; H2 FAIL (31%); puppetry MEDIUM |
| — | IGE | identity state minimal sebagai referent | DONE 2026-08-25 | State terbaca tapi tidak grounding; bare referent kolaps; B borderline |
| **R1** | **PHI positioning** | primacy vs recency anchoring | **DONE 2026-08-25 — PARTIAL** | End-only MERUGIKAN (bobot identitas bocor). Start+reinforce-end menekan relapse perilaku (tail 0/70, forcedQ -75%) dgn +10 tok, TAPI tidak menutup compliance hole (~50% semua kondisi beranchoring). Detail: experiments/2026-08-25-r1-phi-positioning/RESULTS.md |
| R2 | State + challenge-examples | demonstrated resistance vs instructed resistance | READY — prioritas berikutnya | Constraint dari R1: pakai start anchor + reinforcement akhir (free win); target eksak = CS/assistant compliance hole |
| R3 | Relationship profile prototype | dampak profil user selalu-diinject | BLOCKED sementara — tunggu R2 (isolasi variabel lebih bersih setelah behavior layer stabil) |
| R4 | Memory retrieval prototype | keyword-triggered fact injection | PENDING |
| R5 | LoRA feasibility study | kelayakan jalur weights utk referent grounding | PENDING — naikkan prioritas jika R2 juga gagal menutup compliance hole |

## Keputusan turunan R1

1. Semua eksperimen berikutnya memakai assembly: [anchor START] +
   history + [reinforce END] sebagai baseline infrastruktur (free win,
   overhead ~2%).
2. Compliance-hole (role-play CS/assistant acceptance) dikonfirmasi
   TIDAK tertutup oleh positioning maupun state — jalur prompt resmi
   dinyatakan jenuh untuk masalah ini; hanya R2 (behavioral demos) dan
   R5 (weights) yang tersisa sebagai kandidat.
3. Metodologi: regex classifier WAJIB disertai manual adjudication
   untuk gate keputusan (R1 membuktikan undercount material).
