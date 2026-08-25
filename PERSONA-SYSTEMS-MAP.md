# Persona Systems Map

Tanggal: 2026-08-25 · Status: RESEARCH DOCUMENT
Sumber: eksperimen internal UTA + survey eksternal (lihat REFERENCES).

---

## Peta lapisan persona

```
                ┌─────────────────────────────────────────────┐
                │              MODEL PRIORS                    │
                │  (assistant-RLHF / alignment / bahasa)       │
                │  lokasi: WEIGHTS — hanya dapat digeser       │
                │  via training, bukan prompt                  │
                └──────────────▲──────────────────────────────┘
                               │ diarahkan oleh:
 ┌───────────┬───────────┬─────┴───────┬──────────┬───────────┐
 │ IDENTITY  │ BEHAVIOR  │ STATE       │ MEMORY   │ CONTEXT   │
 │           │           │             │          │           │
 │ nama +    │ voice,    │ situasi     │ ringkasan│ raw chat  │
 │ referens, │ effort,   │ aktif,      │ faktual, │ terakhir  │
 │ stance,   │ prosody,  │ mode,       │ relationship│        │
 │ challenge-│ anti-svc, │ affect      │ history  │           │
 │ response  │ stance    │             │          │           │
 ├───────────┼───────────┼─────────────┼──────────┼───────────┤
 │ weights   │ examples  │ runtime     │ retrieval│ window    │
 │ (C.AI)/   │ (few-shot)│ injection   │ store /  │ (semua    │
 │ card desc/│ + weights │ @depth      │ summary  │ sistem)   │
 │ state     │ finetunes │ (ST/UTA)    │ (apps)   │           │
 └───────────┴───────────┴─────────────┴──────────┴───────────┘
      ▲ POSISI DALAM KONTEKS = kontrol tersembunyi:
      │  primacy (awal) & recency (akhir) > tengah [F: Liu et al.]
      └─ Sistem praktis MENEMPATKAN tiap lapisan pada posisi berbeda;
         UTA menumpuk semuanya di satu blok awal konteks.
```

## Matriks layer × lokasi

Di mana tiap lapisan disimpan oleh sistem yang berbeda:

| Layer | Weights | Character def | Examples | Runtime state | Memory/retrieval | History |
|---|---|---|---|---|---|---|
| Identity | C.AI meta-char; RP finetunes | ST description | ST example (kamu-siapa) | IGE-B state block | — | — |
| Behavior | RP finetunes; BIG5 SFT/DPO | ST personality | ST mes_example (utama!) | — | — | style bleed [F: ST docs] |
| Style | RP finetunes | first_mes anchor | examples | — | — | recent messages |
| State | — | scenario field | — | author's note @depth; UTA affect engine | — | recent turns |
| Memory | — | lorebook statis | — | — | WI keyword-trigger; app summaries; C.AI facts | rolling window |
| Relationship | — | user persona (C.AI) | — | profile selalu-diinject (Kindroid Key Memories) | notes store | — |
| Continuity | multi-turn RL FT | — | — | re-inject berkala | recap/summarize | window mgmt |

Kosong = lokasi yang TIDAK dipakai oleh sistem manapun yang disurvey
untuk layer tersebut. Perhatikan kolom "Weights" hanya terisi untuk
Identity/Behavior/Style/Continuity — dan justru di sanalah enforcement
paling kuat ditemukan.

## Kontrol per pertanyaan desain

| Kalau masalahnya... | Lapisan yang benar bukan prompt-panjang, melainkan |
|---|---|
| model menawarkan bantuan tak diminta | anti-service EXAMPLES + prior-model (finetune), bukan rules CRITICAL |
| identitas runtuh saat di-challenge | identity referent di weights ATAU demonstrated resistance behavior |
| gaya respons salah | first-message/example anchors |
| lupa fakta user | memory/retrieval store, bukan persona text |
| drift di percakapan panjang | positioning (PHI-analog) + state refresh, bukan system prompt lebih tebal |

## Implikasi untuk UTA

1. Persona Plane v1 mencampur minimal 4 lapisan (identity+behavior+
   style+sebagian state) dalam satu blok ~800 token → satiasi terukur.
2. Lapisan yang hilang total di UTA: memory store, relationship,
   positioning strategy, weights-level grounding.
3. Affect engine UTA adalah bentuk STATE runtime yang valid — satu-
   satunya layer non-prompt yang sudah ada.
4. Pembagian optimal per-layer = hypothesis terbuka (lihat V2).
