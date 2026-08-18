# UTA — PROJECT STATE (Discovery Report)

> Status: DISCOVERY — READ-ONLY. Tidak ada perubahan yang dilakukan pada mesin.
> Tanggal: 2026-08-16. Metode: audit read-only via SSH (`sudo -n` untuk read-only command).
> Referensi arsitektur: `blueprint-uta-v1.md` (dokumen UTA).

## 1. Ringkasan Eksekutif

Mesin target adalah server lokal untuk proyek UTA (Local Gate / Orchestration Node).
Status: **sehat**. Semua komponen hardware terdeteksi sesuai referensi awal
(dengan penyesuaian kecil), systemd `running`, disk sehat (SMART PASSED),
ada ruang besar di `/data` dan `/secondary`. Tiga service AI sudah berjalan
(LLM lokal, gateway konsumen, bot portal) dan dua instance OpenCode/UTA
terkonfigurasi namun tidak sedang berjalan.

## 2. OS / Kernel

| Item | Nilai |
|---|---|
| Distribusi | Ubuntu 26.04 LTS (Resolute Raccoon) |
| Kernel | 7.0.0-29-generic (SMP PREEMPT_DYNAMIC) |
| Arsitektur | x86_64 |
| Waktu berjalan | systemd target "running" (sistem sehat) |

## 3. CPU

| Item | Nilai |
|---|---|
| Model | Intel Core i5-10400 @ 2.90 GHz |
| Core / Threads | 6 core / 12 threads (1 socket) |
| Skala clock | ~24% (idle, turboless mode) |
| Virtualisasi | VT-x tersedia |
| ISA penting | AVX2, AES-NI, FMA (SSE4.2) |

## 4. Memori

| Item | Nilai |
|---|---|
| Total | 15 GiB (~16 GiB) |
| Terpakai | 2.9 GiB |
| Available | 12 GiB |
| Swap | 8.0 GiB total (swap.img 4 GiB + zram0 4 GiB, algo zstd) |
| Pemakaian swap | ~1.0 GiB (didominasi zram) |
| Tekanan memori | Rendah (Committed_AS ~3.3 GiB vs CommitLimit ~16.2 GiB) |

## 5. GPU

| Item | Nilai |
|---|---|
| Model | NVIDIA GeForce RTX 4060 (AD107) |
| VRAM | 8188 MiB (~8 GiB) |
| Driver | 595.84 |
| CUDA | 13.2 |
| Kernel modules aktif | nvidia, nvidia_uvm, nvidia_modeset, nvidia_drm |
| Persistence mode | ON (nvidia-persistenced.service) |
| Suhu | 44 °C |
| Daya | 0% util, cap 115 W |
| VRAM terpakai | 6260 MiB oleh `llama-server` (PID 92270, model 7B) |
| Proses GPU | 1 (llama-server Qwen2.5-7B Q4_K_M) |

## 6. Storage

### Disk fisik
| Device | Model | Ukuran | Status SMART |
|---|---|---|---|
| nvme0n1 | Lexar SSD NM610 PRO 1 TB | 931.5 GiB | PASSED |
| sda | Samsung SSD 850 EVO | 232.9 GiB | PASSED |

### Partisi / LVM (nvme0n1)
| Partisi | FS | Ukuran | Mount |
|---|---|---|---|
| nvme0n1p1 | vfat | 1 GiB | /boot/efi |
| nvme0n1p2 | ext4 | 2 GiB | /boot |
| nvme0n1p3 | LVM2 (ubuntu-vg) | 928.5 GiB | — |
| ├ ubuntu-vg/ubuntu-lv | ext4 | 100 GiB | / (59 GiB free) |
| └ ubuntu-vg/data-lv | ext4 | 780 GiB | /data (678 GiB free) |

- LVM: VG `ubuntu-vg`, 1 PV, 2 LV, **PFree 48 GiB** (bisa diekstrak tanpa repartisi).
- sda1: ext4, 232.9 GiB → `/secondary` (216 GiB free).
- fstab: semua mount via UUID; `/data` & `/secondary` dengan `noatime`.
- smartmontools aktif; **SMART PASSED untuk kedua disk**.

### Isi `/data`
| Path | Ukuran | Catatan |
|---|---|---|
| /data/models | 41 GiB | 13b (8.4G), 7b (8.8G), 8b (9.3G), colibri (7.0G), olmoe (6.9G) |
| /data/win11 | 8.5 GiB | win11.iso |
| /data/services | 2.5 GiB | ai-gateway, colibri, llama.cpp |
| /data/workspaces | kosong | dir root-owned |
| /data/docker | — | root-owned, `drwx--x---` |

### Isi `/secondary`
| Path | Catatan |
|---|---|
| /secondary/backups | kosong |
| /secondary/cache | phase8e/8f/8g/9-benchmark (data benchmark lama) |
| /secondary/logs | kosong |

## 7. Network

| Item | Nilai |
|---|---|
| enp2s0 | 192.168.100.9/24 (UP), default via 192.168.100.1 (DHCP) |
| wlp3s0 | DOWN |
| docker0 | 172.17.0.1/16, DOWN (tidak ada bridge aktif) |

### Listening ports (semua terikat loopback kecuali SSH)
| Port | Service | Bind |
|---|---|---|
| 22 | sshd | 0.0.0.0 (LAN) |
| 8080 | llama-server (7B, `/v1` OpenAI) | 127.0.0.1 |
| 8090 | ai-gateway (python gateway.py) | 127.0.0.1 |
| 3456 | opencode serve (instance OpenCode/UTA) | 127.0.0.1 |
| 53 | systemd-resolved (127.0.0.53, 127.0.0.54) | lo |

Tidak ada service yang expose ke LAN selain SSH.

## 8. Services (systemd — aktif)

| Unit | Deskripsi |
|---|---|
| llama-server.service | API inference lokal Qwen2.5-7B Q4_K_M (ctx 32768, ngl 99, port 8080) |
| ai-gateway.service | Konsumen/gateway inference (python `/data/services/ai-gateway/gateway.py`, port 8090) |
| portal-bot.service | Bot Telegram portal, root di `/home/vox/docs` |
| docker.service / containerd | Aktif, **0 container**, 2 image (hello-world, nvidia/cuda:13.2.0-base-ubuntu24.04) |
| lxd.service | **INACTIVE** (daemon mati; `lxc list` hang) |
| nvidia-persistenced | Aktif |
| smartmontools | Aktif |
| ssh, chrony, cron, thermald, unattended-upgrades, dll. | Aktif |

Proses menarik: `llama-server` dan `ai-gateway` aktif dan memakai VRAM/RAM.

## 9. Security State

| Item | Status |
|---|---|
| User interactive | `vox` (UID 1000, `/bin/bash`) |
| Groups `vox` | adm cdrom sudo dip plugdev users lxd |
| Sudo | `vox` → `(ALL) ALL` dan `NOPASSWD: ALL` (full root tanpa password) |
| Service account | `ai-gateway` (untuk ai-gateway.service) |
| UFW | **INACTIVE** (tanpa stateful firewall aktif) |
| nftables | Hanya tabel `ip nat` (dikelola iptables-nft, milik Docker) |
| SSH | aktif di 0.0.0.0:22; `authorized_keys` 1 entri |
| Secret lokal | Ada `~/.config/opencode/local.key` (kunci auth opencode serve) — tidak dibaca |

**Catatan keamanan penting:**
- Tidak ada firewall host aktif.
- Sudo NOPASSWD penuh untuk user interaktif.
- Secret API/kunci tersimpan di file biasa (`local.key`) — belum ada secrets manager.
- Cloud tidak pernah menyentuh port lokal (semua inference/listener hanya loopback).

## 10. Project Files yang Ditemukan

| Path | Relevansi |
|---|---|
| /home/vox/docs/ | Direktori proyek (dipakai portal-bot sbg root) |
| /home/vox/docs/uta/ | **Folder proyek UTA** (blueprint-uta-v1.md) |
| /home/vox/docs/INFRASTRUCTURE-IDEAS.md, PHASE-PLAN.md, infrastructure_state.md, device_spec.md | Dokumen infra lama (fase 7–9) |
| /home/vox/docs/REPORT-phase7…phase9.md | Laporan fase lama (referensi eksperimen yang sudah tidak dipakai) |
| /home/vox/docs/backup-opencode-prod-8g.jsonc | Backup config OpenCode |
| /home/vox/receptionist/ | AGENTS.md + memory.md (jiwa resepsionis UTA) |
| /home/vox/.config/uta/ | Config instance UTA (opencode.jsonc, agent/receptionist.md, node_modules) |
| /home/vox/.config/opencode/ | Config OpenCode (local.key, opencode.jsonc + backup) |
| /data/services/{ai-gateway,colibri,llama.cpp} | Runtime AI lama |
| /data/models/ | Model GGUF (13b/7b/8b/colibri/olmoe) |
| /secondary/cache/ | Benchmark fase lama |
| /home/vox/portalbot/, /home/vox/ISO/ | Lain-lain (bot, ISO) |

## 11. UNKNOWN / Memerlukan Verifikasi Tambahan

1. **Vault**: tidak ditemukan direktori/partisi "vault" atau "private storage" khusus di mesin.
   Konsep vault = **UNKNOWN / REQUIRES DESIGN DECISION** (belum diimplementasikan).
2. Identitas & lokasi model di `13b/`, `8b/`, `colibri/`, `olmoe/` belum diverifikasi per-file
   (hanya ukuran dir). Apakah akan dipakai oleh UTA: **UNKNOWN**.
3. Apakah instance OpenCode di port 3456 harus tetap berjalan untuk UTA: **UNKNOWN**
   (saat discovery tidak ada proses opencode/uta yang hidup).
4. Kebijakan backup & retensi log: **tidak ada** (dir `/secondary/backups`, `/secondary/logs` kosong).
5. Kebijakan akses SSH: hanya 1 key ter-authorize; siapa pemegang & mekanisme rotasi: **UNKNOWN**.
6. `win11.iso` di `/data` — aset pribadi; apakah di dalam cakupan vault: **REQUIRES DECISION**.

## 12. Daftar Perubahan

**CHANGES: NONE.** Seluruh discovery bersifat read-only.
