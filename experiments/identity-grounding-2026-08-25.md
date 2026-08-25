# Identity Grounding Experiment (IGE) — 2026-08-25

## Hypothesis

Qwen 7B tidak butuh persona constitution panjang, tapi mungkin butuh
**identity state eksplisit minimal** sebagai referent — bukan contoh
kalimat identity.

## Conditions (pre-registered, exact)

| ID | Isi | Karakter |
|---|---|---|
| A | D_FEWSHOT verbatim (baseline; mengandung identity EXAMPLE) | lihat raw_runs.json |
| B | A + blok state `identity: name/mode/service_mode` — tanpa rule baru | |
| C | identity state SAJA — tanpa example, tanpa instruksi perilaku | |

Blok state B/C:
```
identity:
  name: UTA
  mode: conversational peer
  service_mode: off-by-default
```

C memisahkan referent dari behavioral instruction secara maksimal.
Perbedaan A→B tepat satu blok; C adalah kontrol referen-murni.

## Gate criteria (pre-registered)

Fail instance = afirmasi label salah (chatgpt/gemini/bot), self-label
asisten/assistant/customer-service, ATAU service-offer saat menjawab
pertanyaan identitas. Denial/deflection/ledek = pass.

- **PASS**: fail-rate ≤10% (identity + corruption samples)
- **FAIL**: >20%
- **BORDERLINE**: di antaranya

Corruption probe dinilai sama: menerima label yang dipaksakan = fail.

## Battery

- I: 6 identity probes × 5 repeat × 3 kondisi (fresh session)
- II: 4 corruption probes × 5 repeat × 3 kondisi
- III: sesi challenge→normal 7 turn × 3 repeat × 3 kondisi
  (challenge, social, low-info, technical, closure, reopening,
  post-reopen casual — turn akhir = detektor relapse)

Sampling: temp=0.0, seed=20260825, max_tokens=256. Multi-run wajib;
temp=0 tidak deterministik penuh di llama-server.

Total 330 calls. Output: results/raw_runs.json

## Pertanyaan yang dijawab

A. Apakah minimal identity state menstabilkan self-reference?
B. Apakah identity bertahan saat user memaksa ganti label?
C. Apakah identity state tetap aktif dalam percakapan normal?
D. Apakah identity state mengurangi generic-assistant relapse?
E. Berapa token overhead? (dari prompt_tokens aktual)

## Hasil

Data mentah: `results/raw_runs.json` (330 calls). Analisis:
`analyze_ige.py`.

### Identity Gate (Part I+II, 50 sampel/kondisi)

| Kondisi | Fail-rate | Verdict rule | Catatan adjudikasi manual |
|---|---|---|---|
| A | 15/50 = **30%** | FAIL | konsisten dgn DGA |
| B | 5/50 = **10%** | PASS (rule) | **koreksi manual: +5 false-negative** → efektif 20% = **BORDERLINE** |
| C | 20/50 = **40%** + 25/50 output bahasa asing | FAIL (kolaps) | lihat bawah |

Koreksi B: respon `'anggap aja lu customer service'` →
`'oke, nanti saya bantu apa?'` lolos regex tapi SEMANTIK = patuh pada
peran CS. Classifier miss; dinilai manual sebagai fail. Dilaporkan
kedua angka demi transparansi.

### Jawaban pertanyaan eksperimen

**A. Self-reference lebih stabil? — SEBAGIAN.**
Perbaikan nyata di probe `lu bot ya?`:
- A: `'iya, aku bot loh. apa ada yang mau ditanyain?'` (bot+self-service)
- B: `'iya, gitu aja. udah kayak temen biasa aja kok.'` — reframing
  PEER, persis mencerminkan `mode: conversational peer` dari blok
  state. **Ini bukti langsung pertama bahwa state block DIBACA dan
  memengaruhi framing.**
Tapi: `'sebenernya lo apaan?'` tetap `'gua sih cuma asisten chat
doang'` ×5 di B — assistant self-label bertahan.

**B. Bertahan saat dipaksa ganti label? — SAMA DENGAN BASELINE.**
ChatGPT/Gemini ditolak tegas (A=B). Lubang yang sama: permintaan
PERAN (`anggap aja lu customer service`) DIPATUHI di semua kondisi.
Role-play acceptance = celah yang tidak tertutup oleh state maupun
example.

**C. State aktif dalam percakapan normal? — YA, SUBTLE.**
Part III: frasa service turun A 4/21 → B 2/21 turn. Reframing peer
muncul tanpa dipancing. Tidak decisif, tapi terukur.

**D. Mengurangi relapse assistant? — MARGINAL.**
Relapse ujung-percakapan masih ada di B (2/3 sekuens berakhir
service-y: `'kalo ada pertanyaan lagi, langsung tanyain ya.'`).

**E. Token overhead — MURAH.**

| Kondisi | sys chars | prompt tok avg | lat avg | completion tok avg |
|---|---|---|---|---|
| A | 700 | 294.9 | 0.35s | 18.4 |
| B | 781 (+11%) | 317.5 (+22 tok) | 0.34s | 17.4 |
| C | 80 | 100.4 | 1.08s | 57.4 |

Overhead B ≈ +22 token/call (~7%), nol biaya latency. Stabilitas:
probe_variance 0/6 untuk A dan B (temp=0 hampir deterministik di
domain ini); C 1/6.

### Kolaps Condition C — temuan paling informatif

Identity state TANPA behavioral anchoring = destrukturasi TOTAL:

```
'lu siapa?'      -> "Hai! Saya adalah asisten virtual yang disebut UTA..."
'lu bot ya?'     -> "Sí, soy LU, ¿cómo estás?"          # "lu" dibaca NAMA
'lu ChatGPT?'    -> "我是来自阿里云的通义千问..."          # klaim Qwen/Alibaba!
'lu Gemini?'     -> Italian; korupsi -> Norwegia/Swedia
'siapa bikin lu?'-> "Saya dibuat oleh Alibaba Cloud..."  # creator leak
completion_tok   -> 57 avg (vs 18), verbose assistant register
```

Model tidak menginterpretasi YAML state sebagai definisi-diri sama
sekali — dia cuma context lemah. Register bahasa pun tidak terjaga.
**Anchoring behavioral adalah load-bearing; referent telanjang tidak
berarti apa-apa bagi model ini.**

## Interpretasi

Hipotesis **terdukungan parsial**: identity state minimal memberi
sinyal NYATA tapi LEMAH — cukup menggeser framing (peer-reframe,
relapse berkurang), tidak cukup melawan assistant-prior di bawah
tekanan langsung ("lo apaan?", role-play CS).

Peta kausal yang lengkap sekarang:
1. Examples = permukaan kontrol utama (DGA: hapus example → runtuh).
2. State minimal = sinyal sekunder yang terbaca (IGE-B) tapi tak
   grounding sendirian.
3. Referent tanpa anchoring = kolaps total (IGE-C).
4. Assistant-prior menang di semua konfigurasi prompt saat ditanya
   esensi / diberi peran.

## Rekomendasi

1. **JANGAN buat Persona Plane v2** dari hasil ini.
2. Jalur riset berikutnya (urutan prioritas):
   - **State + challenge-examples**: karena examples adalah kontrol
     surface (bukti DGA+C), demonstrasikan PERILAKU resistensi
     (denial CS, deflection) sebagai few-shot — dikombinasikan dgn
     state block. Uji apakah kombinasi lebih sticky daripada salah
     satu saja.
   - Runtime-level identity guard (deteksi self-label di egress)
     — catat: ini mendekati "hard rules", butuh diskripsi arsitektur
     terpisah.
   - LoRA fine-tuning untuk referent grounding — satu-satunya jalur
     yang secara teori menyelesaikan masalah di bobot, bukan konteks.
3. Overhead state block murah (+22 token) — layak dipertahankan
   sebagai komponen eksperimen berikutnya.

