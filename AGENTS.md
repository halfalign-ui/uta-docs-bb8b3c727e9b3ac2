# AGENTS.md — UTA Project

> Peraturan ini berlaku untuk SEMUA sesi kerja di proyek UTA.
> Setiap agen WAJIB mengikuti peraturan di bawah ini.

## Aturan Utama

### 1. SEMUA pekerjaan dilakukan di vox-space
- vox-space = `192.168.100.9` (Ubuntu 26.04, user vox, ssh alias `vox-space`)
- vox = `192.168.100.42` (lokal, Debian 13) — HANYA untuk push GitHub, bukan untuk coding
- Gunakan `ssh vox-space` untuk semua operasi (edit, test, commit)
- Jika vox-space offline: `sudo /usr/local/bin/boot-vox-space.sh 180`
- `export HOME=/root` sebelum `git push` / `gh` di vox-space

### 2. Dokumentasi di repo
- Docs: `/srv/dev/UTA/docs/` (in repo)
- Setiap fase yang selesai → commit `F{N}-REPORT.md` + update `UTA-PLAN.md` ke repo docs/
- Format laporan:
  - Commit hash, tanggal, jumlah test
  - Ringkasan arsitektur (diagram ASCII jika memungkinkan)
  - File yang ditambah/diubah
  - Security properties
  - Test coverage table
  - Bug fixes (jika ada)
  - Apa yang TIDAK termasuk (deferred)

### 3. Dokumentasi WAJIB di-sync ke GitHub public docs repo
- Public docs repo: `halfalign-ui/uta-docs-bb8b3c727e9b3ac2`
- Clone local: `/tmp/uta-docs-dump/` (vox-space)
- Setelah update docs di repo utama, WAJIB sync ke public docs repo:
  ```
  export HOME=/root
  cd /tmp/uta-docs-dump
  scp vox-space:/srv/dev/UTA/docs/*.md .
  git add -A && git commit -m "docs: update {F{N}-REPORT/...}" && git push origin main
  ```
- Public docs = source of truth untuk ChatGPT GitHub plugin
- Jangan push source code ke public docs repo, HANYA dokumentasi (*.md)

### 4. Struktur test
- Semua test di `gate/tests/`
- Run: `ssh vox-space "cd /srv/dev/UTA/gate && .venv/bin/pytest tests/ -v --tb=short"`
- Setiap fase menambah test baru; test lama tidak boleh gagal (regression)
- Target: semua test pass sebelum commit

### 5. Commit convention
- Format: `F{N}: {Deskripsi} — {N} tests pass`
- Commit di vox-space, push dari vox (lokal) via GitHub token
- Jangan force push kecuali ada alasan kuat

### 6. Checklist sesi kerja
Sebelum mulai:
1. `ssh vox-space` — pastikan online
2. `cd /srv/dev/UTA/gate && .venv/bin/pytest tests/ --tb=no -q` — baseline test
3. Baca `docs/UTA-PLAN.md` untuk status terkini

Sesudah selesai:
1. Run semua test → pastikan pass
2. Commit di vox-space
3. Push ke GitHub
4. Update docs di `docs/` (in repo) + sync ke public docs repo
5. Update `docs/UTA-PLAN.md`

## Infrastruktur

| Item | Lokasi |
|------|--------|
| Source code | `/srv/dev/UTA/gate/` (vox-space) |
| Virtualenv | `/srv/dev/UTA/gate/.venv/` (Python 3.14.4) |
| Vault dir | `/data/vault/` (vault:vault, 0700) |
| Cache/secondary | `/secondary/` (229G, jangan GANGGU benchmark data di cache/) |
| Docs | `/srv/dev/UTA/docs/` (in repo) |
| GitHub (main) | `halfalign-ui/UTA` |
| GitHub (public docs) | `halfalign-ui/uta-docs-bb8b3c727e9b3ac2` |

## Tahap Selesai

| Fase | Status | Test |
|------|--------|------|
| F0/F1 | Discovery + Architecture | — |
| F2 | Local Gate core + guardrail | 70 |
| F3 | MCP tool boundary | +49 = 119 |
| F4 | Vault & secret handling | +29 = 148 |
| F5 | Cloud execution & agent runtime | +61 = 209 |
| F6 | Heartbeat & background | 304 |
| F7 | Model kecil (opsional) | PENDING |
| F8 | Hardening & deploy | PENDING |
