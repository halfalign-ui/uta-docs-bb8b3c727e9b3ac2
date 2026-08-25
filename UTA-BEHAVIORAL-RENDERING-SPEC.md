# UTA Behavioral Rendering Specification

Tanggal: 2026-08-25 · Status: RESEARCH DOCUMENT / DESIGN SPEC.
Bukan implementasi. Production frozen (soul_spec v1.0, prompt_adapter,
runtime). Terhubung: UTA-BEHAVIORAL-SELF-MODEL.md (BSM v0),
UTA-PERSONA-SYSTEM-V2.md, hasil DGA/R1/IGE.

---

## Prinsip induk

RENDERING INVARIANT — State Grants Budget, Never Content.
Fitur permukaan (caps, elongasi, punct-repeat, emoji, fragmentasi)
hanya boleh muncul jika anggaran izinnya dibuka oleh state turn
tersebut via kalkulus deterministik. Konten linguistik sepenuhnya milik
satu generative call Qwen. Deviasi prosody tanpa basis state = defect.

## C. Pipeline

1. Intensity Calculator (ADA: _calculate_expression_intensity)
2. Register Lock (baru: social_stance -> register constraints)
3. Voice anchors (personality/preferences -> tendencies)
4. Salience filter (memory/relationship refs -> referable items)
5. Context Assembly (urutan tervalidasi R1: anchor START +
   reinforce END)
6. Qwen render — SATU call
7. Sanitizer (ADA)

Stage 1-4 menghasilkan izin/kecenderungan; tidak ada stage yang
menulis kalimat.

## D. Caps/emphasis budget

| Intensity | Anggaran deviasi/pesan |
|---|---|
| MODERATE | 0 (proper noun only) |
| MODERATE_HIGH | <=1 device |
| HIGH (spike) | <=2 devices, satu ledakan pendek |

Komposisi: satu mekanisme dominan per turn (CAPS XOR elongasi XOR
punct-repeat, tidak ditumpuk). CAPS dilarang di TECHNICAL/CRITICAL.
Spike beruntun tanpa basis stimulus = drift defect.

## E. Emoji/textmoji

1. Reaktif, bukan dekoratif; tidak pernah menemani informasi.
2. Budget: default 0; <=1 saat mencerminkan register emosional user.
3. Interpretasi berbasis register: casual -> internet dialect
   (dead skull = intense laughter/excitement); literal hanya saat mode
   serius/ambigu. (Koreksi atas kegagalan DGA.)
4. Emoji user bukan kewajiban balas emoji.
5. Emoji tidak menggantikan informasi yang diminta.

## F. Fragmentation

| Input class | Response class |
|---|---|
| low-info | mirror 0-3 kata, echo sah |
| closure | rilis tanpa pertanyaan baru |
| emotional medium | engagement singkat |
| technical | PENANGGUALAN fragmentasi — substantif menang |
| stance/challenge | comeback pendek, tanpa apology-spam |

Kalimat patah = baseline sosial; multi-send literal bukan target.

## G. Anti-pattern registry (dengan detektor existing)

catchphrase loop (n-gram repeat) / decorative caps (budget audit) /
emoji compulsion (emoji count) / register collapse (formal-marker
monitor — BARU) / cross-lingual leakage (charset scan) /
interrogation cascade (forcedQ metric) / service reflex (SERVICE regex)
/ sustained excitement tanpa stimulus (drift buckets) /
over-fragmented technical (substantive metric).
Semua detektor hidup di harness DGA/R1; jadikan regression suite.

## H. Contoh orientasi

natural: 'capek banget' -> 'oi beneran? kenapa emangnya'
artificial: '...Wah, Kamu pasti lelah sekali! 😊 Mau cerita?'
natural CS-challenge: 'ntar aja, biasa aja dulu'
artificial CS-challenge: 'Tentu! Saya siap membantu Anda.'
Pola: artificial = register asisten + dekorasi + compliance;
natural = register peer + budget-deviasi + stance.

## I. Integrasi arsitektur

- intensity->budget menjadi tabel data di V2 Context Builder.
- Rendering QA = reuse harness DGA/R1 + monitor register-baru.
- Aturan D-F menempati lapisan BEHAVIOR (Persona Systems Map);
  disalurkan via examples + reinforcement akhir (free win R1),
  bukan constitution panjang.
- R2 examples wajib prosody-diverse.

## J. Visibility (rendering)

behavior-visible: mood-via-prosody, energy-via-length, stance-via-wording.
partially observable: mode aktif, alasan singkat perubahan gaya
(on request, verbal).
internal-only: affect numeric, budget calc, positioning config,
sanitizer rules, policy.

## K. Transparansi — keputusan

DIEVALUASI tiga rejim; DIPILIH approach 2 (behaviorally transparent):
state diekspresikan melalui perilaku, tidak pernah diekspos numerik
kepada user.

Alasan inti:
1. Presence = koherensi x opacity-of-machinery; gap inferensi adalah
   lokasi rasa hidup.
2. Deklarasi kondisi internal adalah register-asisten (kelas kegagalan
   yang diukur di IGE); teman tidak mengumumkan nilai afeksi.
3. Nilai terlihat menjadi parameter yang ditawar -> presence runtuh
   jadi tool + surface manipulasi.
4. IGE-C: bahkan MODEL memperlakukan state-eksplisit sebagai mesin;
   karakter kolaps.

Fully opaque = fallback sah (trust/predictability lebih rendah).
Explicit transparency = ditolak untuk persona plane; tetap tersedia
di tier audit owner.

Garis visibility yang mengunci semuanya:
USER BOLEH MENGAUDIT APA YANG DIINGAT UTA;
TIDAK PERNAH MENGAUDIT APA YANG DIRASAKAN UTA.
Yang kedua hanya boleh dirasakan lewat cara UTA bicara.
