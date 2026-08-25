# Experiments — Post-F8 Research Track

Track kanonis untuk riset eksperimental pasca-F8. **Tidak ada file di
direktori ini yang di-import oleh production code (`gate/gate/**`).**

## Aturan track

1. Production path (`gate/`) tetap frozen. Eksperimen TIDAK boleh
   mengubah `gate/gate/soul/soul_spec.json` atau `prompt_adapter.py`
   tanpa otorisasi eksplisit.
2. Setiap eksperimen punya direktori bertanggal: `YYYY-MM-DD-<nama>/`.
3. Setiap eksperimen wajib menyimpan:
   - prompt/kondisi EXACT yang diuji (bukan deskripsi),
   - stimulus set,
   - harness/script versi yang menghasilkan data,
   - hasil mentah + ringkasan.
4. Artefak eksperimen dilarang disimpan di `/tmp` — hilang saat reboot.

## Indeks

| Eksperimen | Status | Ringkas |
|---|---|---|
| [2026-08-25-persona-ablation](2026-08-25-persona-ablation/) | DONE | Ablation 4 kondisi prompt vs Qwen2.5-7B; D_FEWSHOT menang; B_CURRENT gagal anti-service & effort matching |
| [2026-08-25-live-harness](2026-08-25-live-harness/) | ACTIVE | HTTP harness port 8889, mode frozen/fewshot, untuk live persona test |
| [2026-08-25-dga](2026-08-25-dga/) | DONE | DGA: H1 PARTIAL, H2 identity gate FAIL (31%), puppetry MEDIUM — retain as research candidate |
| [archive/p0p2-stabilization-baseline.json](archive/p0p2-stabilization-baseline.json) | HISTORICAL | Baseline output era stabilisasi P0/P2 (dipulihkan dari file accidental `gate/-q`) |

## Berikutnya

- Riset referent grounding (identity state / fine-tuning ringan).
  DGA membuktikan jalur tambah-rules/tambah-example sudah jenuh.
