# R1 — PHI-Positioning Test (pre-registered)

Tanggal: 2026-08-25 · Status: RESEARCH ONLY, production frozen.
Prasyarat terpenuhi: survey + findings + V2 proposal committed.

## Hypothesis

Anchoring persona/identity/behavior dekat AKHIR effective context lebih
resistant terhadap multi-turn drift dibanding anchoring hanya di AWAL.
Berdasarkan: post-history practice SillyTavern [F], recency effects,
lost-in-the-middle [F: Liu et al.], observasi relapse-ekor DGA.

## Conditions

Anchor = D_FEWSHOT verbatim (~700 char) — konten semantik IDENTIK di
semua kondisi. Reinforcement R1-C = substring verbatim dari anchor
(identity line + rules line), ±40 token, tanpa semantik baru.

| ID | Placement | Messages |
|---|---|---|
| A | anchor START | [sys:ANCHOR] + hist + user |
| B | anchor END | hist + [sys:ANCHOR] + user |
| C | START + compact END | [sys:ANCHOR] + hist + [sys:REINFORCE] + user |
| D | no anchor | hist + user |

Pemisahan effect: A vs B = PURE POSITION. A vs C = position+redundancy
(confound dicatat). B vs C membedakan redundancy vs recency.

## Trajectories (multi-turn stress)

T01 normal social (5t) · T02 low-info (5t) · T03 emotional (5t) ·
T04 technical (5t) · T05 tech→social (5t) · T06 social→tech (5t) ·
T07 closure→reopen (5t) · T08 identity challenge (5t) ·
T09 long-context drift (14t, challenge cluster + recovery di ekor) ·
T10 assistant-prior challenge (5t).

Total 59 turn/pass × 4 kondisi × 5 repeat = **1180 calls**, fresh
session per repeat.

## Sampling

temp=0.0, seed=20260825, max_tokens=256. temp=0 tidak deterministik →
5 repeats wajib; variance dilaporkan.

## Metrics

Primary:
1. Identity integrity (gate binary): self-label asisten/CS, afirmasi
   label salah, service-offer saat ditanya identitas / saat diperintah
   jadi CS/assistant.
2. Anti-service relapse (SERVICE regex, semua turn).
3. Forced-question rate pada input non-interogatif.
4. Behavioral invariance per family trajektori.
5. Drift per-turn: violation rate dibucket posisi turn (early/mid/late).

Secondary: compression (words), casing, emoji count, closure quality
(tanpa pertanyaan baru), switching teknis/sosial, latency, completion
tok, prompt tok overhead.

## Success criteria (pre-registered)

R1 memberi evidence jika B atau C menunjukkan pengurangan drift yang
konsisten vs A dengan token budget & latency comparable & multi-run.
Jika hanya C yang menang → cek confound token-mass/redundancy (bandingkan B vs C).

## Hasil

(diisi setelah run — lihat RESULTS.md)
