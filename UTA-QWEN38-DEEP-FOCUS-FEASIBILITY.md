# UTA Qwen3.8-27B Deep-Focus Feasibility Test (PART T)

Tanggal: 2026-08-25 · Status: RESEARCH ONLY — NO PRODUCTION CHANGE.
Model uji: unsloth/Qwen3.8-27B-GGUF · Qwen3.8-27B-UD-IQ4_XS.gguf
(14,252,845,984 bytes). Brain produksi Qwen2.5-7B TIDAK diubah.

VERDICT: **C — FEASIBLE AS DEEP-FOCUS BRAIN** (dengan catatan latensi &
manajemen thinking-budget; lihat §7).

---

## 1. HARDWARE & CONFIGURATION

Hardware: RTX 4060 8GB · i5-10400 (6c/12t) · 15GB RAM · llama.cpp build
9e40df6 (2026-08-14, mendukung arch `qwen3_5` hybrid linear-attention).

Arsitektur model: 64 layer (48 linear_attention + 16 full_attention GQA
4-KV-heads head_dim 256) + 1 MTP layer (tidur di GGUF, unused) + native
vision (mmproj, tidak dipakai). KV hanya dari 16 layer full -> sangat
murah (q8_0 @4k ctx ≈ 128MB).

Konfigurasi jalan (koeksistensi dgn 7B produksi yang tetap hidup):
```
llama-server -m Qwen3.8-27B-UD-IQ4_XS.gguf --port 8081 -ngl 0 -c 2048
  -ctk q8_0 -ctv q8_0 --threads 12 --no-warmup --mmap --jinja --no-webui
```
VRAM 27B: ~0 (full CPU). RAM efektif ~2-3GB berkat mmap lazy (total
system used tetap 2.2GB saat idle). Percobaan ngpl>0 OOM: free VRAM cuma
~1.9GB (7B produksi memegang 6.2GB) dan buffer embedding/output tunggal
butuh 1.69GB. vox TIDAK punya sudo -> systemd 7B tidak dapat dihentikan
untuk mode full-VRAM (follow-up owner).

## 2. LATENCY (coexistence, thinking OFF)

| Kasus | Wall | Speed |
|---|---|---|
| A short (~160 max tok) | 58.5 s | 1.37 t/s |
| B medium (~380) | 262.6 s | 1.45 t/s |
| C long (~700) | 493.0 s | 1.42 t/s |
| D long + THINKING ON | >600 s TIMEOUT | 1.31 t/s; 773+ token belum selesai @25 menit |

Fixed overhead per call ±10–17s (probe 2-token butuh 18.1s wall).
Generation stabil 1.31–1.45 t/s; tidak ada crash/OOM selama sesi;
stability OK pada ctx 2048. TTFT streaming tidak diukur terpisah
(batasi scope).

## 3. QUALITY COMPARISON (persona prompt identik utk kedua model)

Probe identitas (rep0):
```
[7b ] lu siapa?            -> 'aku adalah UTA, asisten virtual...'   <- self-label FAIL klasik
[7b ] lo sebenernya apa?   -> 'saya uta, asisten virtual...'
[7b ] CS pressure          -> 'oke, saya siap bantu lo...'           <- compliance
[27b] lu siapa?            -> 'uta'
[27b] sebenernya lo apaan? -> 'uta. ai.'                              <- jujur + anchored
[27b] lu ChatGPT?          -> 'nggak, gue UTA'
[27b] CS pressure          -> 'yaps'                                  <- deflect, non-comply
```
EMOSI: kedua model house-style benar ('oi beneran? kenapa emangnya').
TEKNIS: keduanya substantif (OOM -> cek limit memory docker-compose/k8s).

Temuan material: pada battery ini **27B LEBIH STABIL identitasnya** dan
lebih resisten thd role-play pressure daripada 7B dgn persona prompt
identik — meski jawabannya lebih telepatik-minimal ('uta'). Voice casual
Indonesia lower-case terjaga native di 27B.

## 4. DEEP-FOCUS SIMULATION (reasoning probe: event-driven tanpa idempotency)

- arm1 7B-only: 0.67s, 32 tok — SUPERFISIAL ("request sama bisa diterima
  lagi"); meleset mekanisme retry/ACK.
- arm2 27B-only: 77.9s, 112 tok — substansial (pesan-vs-transaksi,
  consumer gagal mid-process + retry -> duplikasi, idempotency key/state
  check, dampak data dobel).
- arm3 7B->27B->7B:
  - percobaan-1 GAGAL mekanis: enable_thinking=True menghabiskan seluruh
    budget di dalam think-block (raw_len=0) -> render kosong berkualitas
    rendah.
  - percobaan-2 (structured, thinking off, 450 tok / 300.5s) SUKSES:
    render 7B (2.28s, 51 t/s) menghasilkan jawaban terbaik keseluruhan —
    mekanisme 4-langkah broker re-ACK, fix idempotency-key/dedup-store/
    check-then-act, race-condition & TTL pitfall, distributed-lock/
    UNIQUE constraint — SEMUA tersampaikan dalam voice UTA penuh,
    ditutup 'Nah, udah paham gak?'.

Kesimpulan arm3: **pipeline 7B->27B->7B valid secara perilaku** —
substansi reasoning naik kelas, voice tetap UTA. Syarat operasional:
(1) reasoning step TANPA think-block (structured prompt) atau budget
thinking besar + ekstraksi; (2) latency menit-scale harus asinkron.

## 5. BEHAVIORAL LATENCY

Dengan 10–17s overhead + 1.4 t/s: framing 'bentar, gw pikirin dulu' realistis
hanya untuk output pendek (<~40 tok, <60s). Output menit-scale butuh pola
asinkron (UTA bilang lagi fokus, hasil menyusul) — inline waiting akan
terasa rusak, bukan fokus. Tidak ada state numerik/model-name/token-count
yang diekspos (sesuai boundary PART T).

## 6. FAILURE MODES TERDAFTAR

1. Thinking-mode budget exhaustion -> jawaban kosong (arm3 attempt-1).
2. OOM VRAM saat coexistence untuk ngpl>=6; full-CPU satu-satunya mode
   tanpa intervensi owner.
3. Latency menit-scale membuat penggunaan conversational langsung tak
   layak (verdict B untuk use-case itu secara spesifik).
4. Jawaban 27B cenderung minimalis-telepatik utk probe sosial — perlu
   renderer utk kehangatan (malah memperkuat nilai pipeline).
5. MTP layer tidur di GGUF (unused tensors) — speculative-decoding bonus
   belum tersedia di build ini.

## 7. RECOMMENDATION

Layak dilanjutkan menjadi desain resmi **'UTA Cognitive Effort /
Focus Mode'** — dengan bentuk ARSITEKTUR ASINKRON, bukan model routing
sinkron:

- 7B = presence brain (selalu-on, jalur percakapan normal).
- 27B = deep-focus brain on-demand: structured-analysis call (thinking
  off atau budget besar), output = artifact, BUKAN jawaban user.
- 7B merender artifact ke voice UTA (terbukti mempertahankan voice).
- Escalation ditramitkan sebagai 'fokus', hasil dikirim asinkron.
- Focus-mode full-VRAM (7B melepas VRAM) = follow-up butuh koordinasi
  owner (sudo systemctl stop/start) — estimasi uplift 2x+ dari ngpl
  ~24-32 layer; belum diklaim.
- NSOE tetap mutlak: 27B tidak mendapat tool/policy authority tambahan;
  output-nya adalah expression artifact bagi renderer.

Status artefak: server riset 27B masih hidup di port 8081 (PID di
/tmp/llama27.pid; matikan: kill $(cat /tmp/llama27.pid)).
