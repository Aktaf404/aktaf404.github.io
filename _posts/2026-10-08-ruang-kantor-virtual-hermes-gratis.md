---
title: "Ruang: Ganti Kantor Virtual Berbayar Jadi 3D Office Gratis Buat Hermes"
date: 2026-10-08 13:30:00 +0700
categories: [Homelab, AI]
tags: [ruang, hermes, virtual-office, tailscale, funnel, systemd]
image: /assets/img/ruang-3d-office.png
---

> Cerita proyek: gw ganti kantor virtual AI gw yang lisensinya $9.99-$35.99 jadi **Ruang** — 3D office open-source yang baca semua status agent lewat `hermes` CLI. Nol rupiah, nol Docker, read-only.

## Kantor lama: enak tapi bayar

Sebelumnya gw pakai my-virtual-office (Docker, Python). Keren, tapi:

- **Demo dibatasin 3 agent**, panel browser/SMS/cron/model manager dikunci
- Full license **$35.99**, early bird $9.99
- Butuh API server key, socat bridge, recreate container tiap ganti `.env` (env cuma kebaca pas create — jebakan klasik)

Terus gw nemu [Ruang](https://github.com/yugienugraha/ruang): 3D virtual office + mission control yang dibuat khusus buat Hermes Agent. Baca semua via `hermes` CLI, **read-only** — dia gak pernah nyentuh config. Langsung gw pindah.

## Ruang itu apa

[![3D office Ruang](/assets/img/ruang-3d-office.png)](https://github.com/yugienugraha/ruang)

3D office isometrik: agent-nya keliatan duduk di meja kerja, yang idle main ping-pong sama console game di game room. Ada bendera merah putih di depan gedungnya 🇮🇩

Selain 3D-nya, ada mission control lengkap:

[![2D dashboard](/assets/img/ruang-2d-dashboard.png)](https://github.com/yugienugraha/ruang)

- **Kanban tasks**, calendar, activity feed
- **Logs** (agent, gateway, errors) + command log
- **Memory & folders** agent, sessions, channels
- Status cron berikutnya, gateway health, token usage

Semua data dibaca live via `hermes` CLI (`hermes status --all`, `hermes kanban --board default list --json`, dll). UI-nya bisa Bahasa Indonesia.

## Install-nya 1 menit

```bash
curl -fsSL https://raw.githubusercontent.com/yugienugraha/ruang/main/install.sh | bash -s -- --service
loginctl enable-linger root   # biar survive logout
```

Installer-nya ngecek Node.js 20+ (kalau gak ada, dia download sendiri, gak nyentuh sistem), install ke `~/.local/share/ruang`, bikin command `ruang`, dan (opsional `--service`) bikin systemd user service.

Default-nya listen di `127.0.0.1:3001`.

## Jebakan port 3001

Di mini PC gw, `ruang`-nya **exit code 0 diam-diam** tiap kali di-start dari systemd. Log bersih, health check gagal. Ternyata **port 3001 udah dipake portfolio 3D HUD** gw (`oprek1-portfolio.service`).

Ruang gak print error "port in use" yang jelas — dia cuma keluar. Kalau service lo `inactive` tapi gak jelas kenapa, cek dulu:

```bash
ss -tlnp | grep 3001
```

Fix-nya pindah port. Edit `~/.config/systemd/user/ruang.service`:

```ini
ExecStart=/home/user/.local/bin/ruang --port 3005
```

Lalu:

```bash
systemctl --user daemon-reload
systemctl --user restart ruang
```

## Publik via Tailscale Funnel

Ruang bind ke `127.0.0.1` only (aman). Buat akses dari luar, gw expose pakai Funnel di port HTTPS dedicated:

```bash
tailscale funnel --bg --https=10001 http://127.0.0.1:3005
```

Peta funnel gw sekarang: `:443` → dashboard Hermes, `:8443` → mini PC HUD, `:10000` → portfolio, `:10001` → Ruang. Satu service = satu port, gak pake subpath (pernah kena: asset path absolut nyasar ke route lain, halaman kebuka tapi mati).

## Publik atau gak: pertimbangannya

**Kelebihan publik:**
- Akses dari HP mana pun tanpa VPN/client
- Link bisa dibagiin (portfolio, demo)
- Klien/kawan bisa lihat status agent live

**Kekurangan publik:**
- Read-only sih, tapi data tetep keliatan: nama model, session list, isi memory/folders agent
- URL jadi target scan bot
- Kalau Funnel mati, akses luar ikut mati

**Setup gw:** tetep Funnel (gak mau install Tailscale di tiap device), tapi **access code dinyalain**:

```bash
ruang access-code new
```

Dia generate kode random, ditampilkan sekali, dan tiap browser harus masukin kode sebelum bisa lihat apa-apa. Plus privacy mode di Settings yang nge-redact secret di UI.

## Ringkasan

| | Office lama | Ruang |
|---|---|---|
| Biaya | $9.99-$35.99 | Gratis |
| Agent | 3 (demo) | Unlimited |
| Cara baca data | API server + key + bridge | `hermes` CLI |
| Docker | Wajib | Gak perlu |
| Modyfikasi config | Bisa (berisiko) | Read-only |

Install satu baris, service auto-start, Funnel untuk publik, access code untuk keamanan. Kombinasi yang pas buat kantor AI di mini PC.
