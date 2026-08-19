# Prototipe Jiwa L1 — Resepsionis dengan Memory Self-Update

> 2026-08-16. Ditempa di vox (opencode 1.18.5). Terbukti bekerja.
> Status: **prototipe**, belum di-deploy ke vox-space.

## Tujuan
L1 resepsionis harus punya "jiwa": kepribadian + memori yang **konsisten lintas sesi**
dan **update sendiri** (tidak menunggu diedit manual di AGENTS.md) — meniru pola OpenClaw
memory, tapi hemat token (tanpa injeksi 13k char per turn).

## Struktur (3 file)

| File | Isi | Peran |
|---|---|---|
| `~/.config/opencode/agent/receptionist.md` | persona (frontmatter `model` + prompt) | agent custom resepsionis, model cloud |
| `~/receptionist/AGENTS.md` | instruksi jiwa + self-update memory | auto-inject tiap sesi di direktori kerja |
| `~/receptionist/memory.md` | memori jangka panjang (identitas/preferensi/aturan/riwayat) | dibaca & ditulis ulang oleh agent sendiri |

## Cara kerja
1. Sesi baru dibuka di `~/receptionist/` → AGENTS.md otomatis di-injeksi ke context.
2. Instruksi: "WAJIB baca `memory.md` saat mulai percakapan baru" → agent panggil Read.
3. Kalau user mengungkap fakta baru → agent **sendiri** panggil Write `memory.md`
   (tambah/perbarui, tanpa duplikat, max ~30 baris).
4. Sesi berikutnya baca `memory.md` yang sama → identitas & preferensi tetap konsisten.

Kuncinya: **file yang menulis memory adalah agent, bukan manusia.** Ini yang membuatnya
berbeda dari "edit AGENTS.md manual".

## Hasil test (2 sesi terpisah, model gemini-3.5-flash-lite via Cloudflare)

- Sesi 1: user memperkenalkan diri ("panggil gw Bang Ojan", "langsung eksekusi, jangan ribet")
  → resepsionis baca `memory.md`, tulis fakta ke file. ✅
- Sesi 2 (percakapan baru): "gw siapa?" → jawab **"Lu kan Bang Ojan"** (ingat lintas sesi) ✅
  + fakta baru (laporan build ringkas) ditambahkan ke baris yang sama, **tidak duplikat** ✅

Isi memory.md setelah test:
```
# Memori Resepsionis
- identitas_user: Bang Ojan
- preferensi: Suka langsung eksekusi tanpa ribet; laporan build selesai cukup summary pendek.
- aturan_kerja: Jangan ribet, langsung eksekusi permintaan user; ringkas laporan build.
- riwayat_singkat: ...
```

## Biaya vs OpenClaw
- OpenClaw: inject memori penuh tiap turn (~13k char/turn → ~25k token/turn).
- Prototipe ini: injeksi AGENTS.md sekali di awal sesi + Read memory.md sekali.
  Update memory hanya saat ada fakta baru. **Jauh lebih hemat**, esensi jiwa tetap sama.

## Keterbatasan & langkah lanjut
1. Memory ini **linear & berukuran terbatas** (~30 baris) — belum ada search lintas
   percakapan. Kalau butuh memori faktual besar → tambah tool `memory_search` (pola
   OpenClaw `memory.search.rememberAcrossConversations`, `src/claws/schema.ts:235`).
2. Belum dipisahkan siapa yang bisa tulis memory (siapa pun agent di direktori ini).
   Nanti bisa di-lock supaya hanya sesi L1 yang boleh menulis.
3. Belum diuji sebagai bagian dari alur tiket L1→L2 di vox-space (masih test terpisah).
4. Model default (gemini-3.5-flash-lite) cukup untuk persona; saat L1 benar-benar
   jadi orkestrator, mungkin perlu model yang lebih kuat (Claude/Gemini penuh).

## Catatan deploy (saat pindah ke vox-space)
- Copy 3 file di atas ke host vox-space (`~/.config/opencode/agent/receptionist.md`,
  `~/receptionist/{AGENTS.md,memory.md}`).
- Jalankan sesi resepsionis dengan `--dir ~/receptionist` (AGENTS.md auto-inject).
