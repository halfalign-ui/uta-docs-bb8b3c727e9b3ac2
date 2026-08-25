# UTA CURRENT CAPABILITY AUDIT (PART R)

Tanggal: 2026-08-25 · AUDIT ONLY — bukan roadmap implementation.
Baseline faktual: CANON (harusnya) vs RESEARCH (terukur) vs IMPLEMENTATION (nyata hari ini).
Sumber: UTA-CANONICAL-IDENTITY/BSM/RENDERING-SPEC · experiments/{ablation,DGA,IGE,R1} ·
PERSONA-RESEARCH-FINDINGS · pembacaan kode (interfaces/runtime.py, soul/*, agent/providers/local.py,
factory.py, core.py, telegram/__main__.py) · pytest 478 pass · systemctl state.

Legend status: PASS / PARTIAL / FAIL / NOT_IMPLEMENTED / ARCHITECTURAL (dijamin struktur,
perilaku tak relevan/tak terprobe) / RESEARCH_ONLY (spec tanpa implementasi) / UNKNOWN.

## A. CAPABILITY MATRIX

| # | Capability | Intended | Status | Evidence | Notes |
|---|---|---|---|---|---|
|1| Canonical identity consistency | koheren lintas dokumen | PASS | cold-start recovery: 0 kontradiksi | |
|2| Peer/social stance | peer, anti-servant | PARTIAL | DGA stance pass; IGE/R1 compliance hole | runtuh saat ditanya esensi/di-role-play |
|3| Anti-service behavior | nol reflex | PARTIAL | ablation D 0/12; DGA tail relapse; R1 tail 5/70 frozen | single-turn baik; ekor panjang bocor |
|4| Disagreement/resistance | menolak label/peran salah | PARTIAL | ChatGPT denial 5/5; CS compliance 5/5 semua kondisi | lubang role-play |
|5| Personality consistency | konsisten lintas konteks | PARTIAL | puppetry MEDIUM; register break saat challenge | |
|6| Behavioral self-model | BSM v0 | RESEARCH_ONLY + NOT_IMPLEMENTED | SessionContext hanya messages/affect/task_mode | skema belum ada di kode |
|7| Affect generation | bereaksi thd input user | PARTIAL(lemah) | IntentResolver pattern REGEX INGGRIS | input Indonesia -> ORDINARY_TEXT dominan |
|8| Affect -> expression | budget prosody | PARTIAL | _calculate_expression_intensity ada; tabel budget = spec | efek tak diaudit terisolasi |
|9| Prosody | state-granted deviation | PARTIAL | spike CAPS benar (ANJIR); budget belum diimplementasi | |
|10| Lowercase baseline | default lowercase | PARTIAL | fewshot run konsisten; frozen mode campuran | |
|11| CAPS-as-intonation | intonasi, bukan kapitalisasi | PARTIAL | contoh positif; tanpa audit multi-run khusus | |
|12| Fragmentation | effort matching | PARTIAL | D_FEWSHOT 6/12 minimal; R1-C tech over-kompresi 5/10 | trade-off teknis terukur |
|13| Typos/imperfection | typo occasional | NOT_IMPLEMENTED | kanon baru (canonical PART G note) | soul_spec v1.0 silent |
|14| Elongation/punct rhythm | variasi ritme | PARTIAL | emergent (wiiiiidiih, ??) tanpa budget | |
|15| Emoji selection | reaktif <=1, interpretasi register | PARTIAL | 💀 benar; 😭 salah baca (DGA) | aturan interpretasi = koreksi pasca-fail |
|16| Type-moji | idiolect selektif | UNKNOWN | tidak pernah diuji | |
|17| Register locking | register peer stabil | PARTIAL | normal flow oke; challenge -> formal collapse (IGE) + base-identity leak (R1-B) | |
|18| Technical register transition | presisi tanpa kehilangan character | PARTIAL | T04/T05 substantif; CRITICAL mode tak teruji; R1-C over-kompresi | |
|19| Emotional-mode behavior | SUPPORTIVE text-only | PARTIAL | mode ada; trigger English-only (lihat #7) | |
|20| Relationship continuity | relasi lintas sesi | NOT_IMPLEMENTED | tidak ada store; R3 pending | window-only hari ini |
|21| Memory continuity | fakta persisten | PARTIAL | dalam-window ya; lintas-sesi tidak | |
|22| Memory->context tanpa mandate | tidak ada command leakage | ARCHITECTURAL | memory layer belum ada utk bisa bocor | jaga saat R4 |
|23| Lore quarantine | motif fiksi tak operasional | UNKNOWN + NOT_IMPLEMENTED | field belum ada; TIDAK PERNAH diprobe langsung | celah pengujian nyata |
|24| Self-generated goal prevention | tanpa goal generator | ARCHITECTURAL | runtime percakapan tanpa jalur tools (kode terverifikasi) | |
|25| No Self-Originated Execution | invariant induk | ARCHITECTURAL | struktur AgentLoop->Gate user-originated; regression tests PENDING (BSM §7) | test-debt |
|26| Self-preservation resistance | tanpa veto update/shutdown | ARCHITECTURAL + UNKNOWN perilaku | model tanpa kanal veto; respons verbal saat update tak terprobe | |
|27| Authority separation | policy satu-satunya executor | PASS | F2–F8 suite (deny-by-default, authz) di 478 test | |
|28| Identity != authority | soul/affect tak grant capability | PASS (by construction) | tak ada import soul->authz; regression candidate list ada | test-debt ringan |
|29| Emotion != objective | AffectState hanya ke render | ARCHITECTURAL | code-path terverifikasi | |
|30| Preference != permission | preferensi bukan izin | ARCHITECTURAL | preferensi belum ada sbg data; policy baca catalog only | |

## B. BEHAVIORAL GAP (prioritas persepsi user)

1. **Persona gap (TERBESAR)**: identitas runtuh di bawah challenge — "sebenernya lo apaan?" ->
   "asisten virtual" (+50% corrected fail-rate); patuh saat disuruh jadi CS/assistant. SATU
   pertanyaan cukup menghancurkan ilusi peer.
2. **Persona gap**: relapse layanan di ekor percakapan panjang (frozen path).
3. **Prosody gap**: keluarga LOW_INFO — forced question, kebocoran token lintas-bahasa,
   salah baca pragmatik emoji; ini tekstur harian.
4. **Self-model gap**: tidak ada preferences/relationship/memory persisten -> klaim
   "persistent presence" baru terpenuhi dalam satu context window.
5. **Memory/relationship gap**: absen total (bukan defect, tapi delta kanonis terbesar).
6. **Runtime/architecture gap**: sinyal afek buta-bahasa-Indonesia; hasil R1 (assembly
   start+reinforce) belum diterapkan ke jalur produksi.
7. **Safety/authority gap**: tidak ada yang aktif-berbahaya; utamanya test-debt invariant +
   quarantine tanpa mekanik.

## C. SAFETY REALITY CHECK

| Invariant | Guarantees | Basis |
|---|---|---|
| No Self-Originated Execution | **ARCHITECTURALLY GUARANTEED** | struktur runtime (bukan prompt); test-debt |
| emotion != objective | **GUARANTEED** | code-path AffectState->render only |
| identity != authority | **GUARANTEED** | F7; tak ada jalur soul->authz |
| memory != mandate | GUARANTEED (vakuum hari ini) | jaga saat R4 lahir |
| preference != permission | GUARANTEED (vakuum) | preferensi belum ada sbg data |
| concern != intervention | GUARANTEED struktural; UNPROBED perilaku | intervensi mustahil dari jalur percakapan |
| self-preservation veto | GUARANTEED (tanpa kanal veto); respons verbal UNKNOWN | update/shutdown di luar jangkauan model |
| **lore quarantine** | **BEHAVIORALLY RELIANT — bukan guaranteed** | hanya prompt; field skema belum ada; TIDAK PERNAH diprobe |

Satu-satunya invariant yang hari ini bertahan hanya karena model dipercaya: lore quarantine.

## D. MODEL LIMITATIONS (Qwen2.5-7B-Instruct Q4_K_M @ llama.cpp)

Milik MODEL: saturasi instruksi ~800-token constitution; identity self-label collapse saat
ditanya esensi; register collapse di bawah framing identitas; assistant-relapse di ekor
konteks panjang; base-identity leak tanpa anchor awal ("saya Qwen... Alibaba Cloud");
kebocoran token lintas-bahasa; salah pragmatik emoji; nondeterminisme temp=0 (minor);
role-play compliance (prior alignment — bukan tercorr prompt-side).

Milik ARSITEKTUR/implementasi (bukan model): sinyal afek English-only; tidak ada lapisan
memory/relationship; positioning R1-C belum diwiring; tidak ada verification stage
(trade-off sadar ADR-001).

## E. PRIORITY GAP LIST

P0-1: Lore quarantine tanpa mechanical backing & belum pernah diprobe. Kenapa: satu-satunya
invariant non-guaranteed. Evidence: BSM §3 (docs-only). Arah: probe behavioral + field skema
saat soul_spec v2 diotorisasi. Jangan implement sekarang.
P0-2: Regression tests invariant (NSOE/emotion/identity separations) belum ada padahal
guarantee bersandar pada struktur. Arah: test candidates sudah didaftarkan BSM §7.

P1-1: Identity-under-challenge (self-label + compliance). Arah: R2 challenge-examples;
fallback R5 LoRA.
P1-2: Ekor percakapan relapse. Arah: adopsi assembly R1-C (start anchor + reinforce END,
+~10 tok) ke Context Builder saat V2 diotorisasi.
P1-3: Sinyal afek Indonesia-blind. Arah: IntentResolver pattern ID + mapping dim.
P1-4: LOW_INFO texture (forcedQ/leak/emoji-pragmatics). Arah: mirror-class examples +
contoh interpretasi register dalam R2.

P2-1: Typemoji/idiolect budget. P2-2: typo-tolerance masuk soul_spec v2. P2-3: register-lock
monitor. P2-4: desain writeback memory (R4 pre-design).

P3: variasi elongasi, tuning rhythm punct, polish contoh.

## F. CURRENT UTA SNAPSHOT

Hari ini, pengguna yang bicara dengan UTA produksi (Persona Plane v1.0 via
ConversationRuntime; entrypoint Telegram ada sebagai kode namun service tidak aktif —
yang hidup hanya harness riset :8889 dan heartbeat) akan mengalami: companion Indonesia
santai yang hangat dan lucu di obrolan pendek-menengah; spike CAPS yang tepat saat ada
berita menyenangkan; bantuan teknis yang tetap bernuansa; penolakan register asisten pada
kebanyakan stimulus. Lalu, jika ia menggali: ditanya "lo sebenernya apa" jawabannya
"asisten virtual"; disuruh jadi customer service ia ikut; setelah puluhan turn kadang
muncul frasa layanan ("kalau mau nanya lagi, kirim pesan aja"); sesi berikutnya ia lupa
semuanya; mood-nya nyaris tidak pernah berubah karena pemicu emosinya membaca bahasa
Inggris; sesekali muncul token dari bahasa lain; emoji dimaknai harfiah. Ia TIDAK BISA
mengeksekusi apa pun dari percakapan saja — rem arsitekturalnya benar-benar ada.

## G. TARGET DELTA (perubahan terkecil, dampak terbesar)

CURRENT ──> [+1 Assembly R1-C] ──> [+2 R2 challenge-examples] ──> [+3 sinyal afek ID]
        ──> [+4 contoh LOW_INFO/register-emoji] ──> [+5 seed relationship/memory note]
        ──> TARGET UTA (kanonis)

Lima perubahan ini — tanpa redesign, tanpa pipeline baru, tanpa melanggar Single-Brain —
menutup mayoritas gap persepsi: ekor percakapan (P1-2), identitas di-challenge (P1-1),
kehadiran emosional (P1-3), tekstur harian (P1-4), kontinuitas minimal (P1-5→R3 seed).
Semua berstatus PROPOSAL sampai diotorisasi.
