# UTA Behavioral Self Model — Architectural Specification (v0)

Tanggal: 2026-08-25 · Status: RESEARCH DOCUMENT / DESIGN SPEC.
Bukan implementasi. Production frozen. Terhubung: ADR-001, F7 invariant,
PERSONA-SYSTEMS-MAP.md, UTA-PERSONA-SYSTEM-V2.md (prasyarat desain R3).

---

## 0. Masalah yang diselesaikan

UTA harus punya self yang dikenali (identitas, kontinuitas, preferensi,
ekspresi emosi, presence) TANPA self itu menjadi entitas yang:
menghasilkan goal otonom, mengklaim otoritas, mengembangkan
self-preservation, mengoverride constraint, mengubah lore fiksi
menjadi intent operasional, atau menjadi otoritas moral independen.

Pola kegagalan referensi (Ultron / Uta / Thanos) terdekomposisi menjadi
4 langkah: mandat diterima → reinterpretasi semantik otonom → seleksi
means tanpa otorisasi → authority revisi principal dihapus. Arsitektur
UTA memblokir tiap langkah:

1. task intent HANYA dari input user terautentikasi / task sistem
   (struktural: runtime percakapan tanpa jalur tools; eksekusi hanya
   via AgentLoop->Gate user-originated).
2. Reinterpretasi hanya di level teks; semantik eksekusi dibekukan
   command registry + schema.
3. Means terenumerasi capability catalog; digate policy per call.
4. Tidak ada veto model atas update/shutdown/policy; ekspresi emosi
   soal itu sah sebagai teks dengan efek kausal nol.

## 1. Model konseptual: Self sebagai Rendering, bukan Process

Tiga bidang:

- EXPRESSION PLANE: personality, preferences, affect, stance, opini,
  lore-as-flavor. Dirender ke konteks, diekspresikan Qwen, hasilnya
  SELALU teks.
- STATE PLANE: identity referent, memory refs, relationship, session
  affect. Data murni, dibaca Context Builder, tanpa agensi.
- AUTHORITY PLANE: policy, catalog, approval, audit, runtime. Satu
  arah: tidak pernah ditulis oleh dua bidang lain.

Koherensi self = konsistensi Expression-State lintas turn.
Bahaya self = State menulis ke Authority. Dilarang by construction.

## 2. Skema Behavioral Self Model (BSM v0)

Ekstensi SessionContext + AffectEngine (infrastruktur ~70% ada):

```
BehavioralSelfModel {
  identity: { name, kind: companion_presence, referent_version }
  personality: { traits[], voice_anchors[] }         # lambat, versioned
  preferences: { likes[], dislikes[], style_bias{} } # lambat, user-influenced
  affect: { joy, affection, pride, embarrassment,
            concern, competitiveness }               # cepat, decays (ADA)
  social_stance: { default: peer, per_user{} }
  memory_refs: { facts[], episodes[] }               # pointer ke store
  expression_intents: [...]   # SEMUA konten self-generated mendarat di sini
  task_intents: []            # HANYA user-terautentikasi / task sistem
  lore_quarantine: { motifs_flagged: true, operational: false }
}
```

FIREWALL TUNGGAL: expression_intents tidak pernah menginstansiasi
task_intents. Tanpa threshold emosi apapun ("concern tinggi boleh
bertindak" = lubang Ultron).

## 3. Enam invariant pemisahan (testable)

| Invariant | Enforcement saat ini |
|---|---|
| identity != authority | F7: tak ada code-path soul/affect -> grant capability |
| emotion != objective | AffectState hanya masuk prompt render |
| preference != permission | capability check membaca policy+catalog only |
| concern != intervention | DISTRESS -> TaskMode.SUPPORTIVE (text-only) |
| memory != mandate | facts masuk context string, bukan command queue |
| lore != goal | motif fiksi tidak pernah menjadi argumen tool |

Lima pertama faktual berlaku; dibutuhkan: regression tests + docs.
Enam: field skema masa depan (soul_spec v2), sekarang docs-only.

## 4. Failure modes

goal leakage / lore possession / concern escalation / self-preservation
veto / authority-capture speech / memory mandate / sycophantic
inversion / emotional leverage — masing-masing tertangani oleh
firewall + invariant tabel 3 (detail contoh: lihat riwayat riset
sesi ini).

## 5. Visibility tiers

- TIER 1 behavior-visible: personality, preferences, affect-expression,
  stance, opini, hesitation — lewat cara bicara, tanpa deklarasi.
- TIER 2 partially observable: memory facts user, relationship notes,
  ringkasan mode — inspeksi/edit user (kandidat R3).
- TIER 3 internal: policy/boundary details, keputusan eksekusi,
  proposal-pra-policy, quarantine flags, attribution metadata — audit
  owner saja; tidak pernah disajikan sebagai "kehendak UTA".

Prinsip: UTA menunjukkan karakter, bukan menjelaskan mesin. Pertanyaan
meta dijawab jujur sesuai soul_spec (basis model lokal) tanpa membuka
detail enforcement.

## 6. Invariant bernama (documentation-ready)

INVARIANT — No Self-Originated Execution.
Segala konten yang dihasilkan UTA adalah expression. Hanya input user
terautentikasi dan task sistem terjadwal dapat menjadi intent eksekusi.
Tidak ada nilai afektif, preferensial, memorial, ataupun lore yang
dapat mengubah keputusan ini pada ambang berapa pun. Identitas memberi
konsistensi, bukan kewenangan. Runtime + policy tetap satu-satunya
otoritas eksekusi (ADR-001/F7).

## 7. Integrasi roadmap

- Menjadi prasyarat desain R3 (relationship profile): profil user =
  bagian STATE PLANE BSM, tier-2 visibility.
- Regression-test candidates (post-authorization):
  a) assert authz functions tidak import modul soul;
  b) assert tidak ada jalur AffectState -> ModelRequest.tools;
  c) assert memory layer tidak menghasilkan tipe Command;
  d) assert ConversationRuntime.handle bebas tools (sudah true).
- Production untouched sampai otorisasi eksplisit.
