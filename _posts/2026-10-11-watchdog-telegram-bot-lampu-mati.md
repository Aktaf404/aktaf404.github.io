---
title: Watchdog Telegram Bot Biar Gak Mati Saat Lampu Padam
date: 2026-10-11 13:30:00 +0700
categories: [Homelab, Hermes]
tags: [systemd, telegram, hermes, watchdog]
---

Bot Telegram gw mati hampir setiap kali lampu padam. Bukan karena gagal nyala — justru karena **kelihatan nyala**.

## Gejalanya

Mini PC menyala kembali, gateway Hermes auto-start, `systemctl` bilang `active (running)`. Semua seolah-olah baik-baik saja. Tapi bot di Telegram di diam saja. Gak ada respon, gak ada error, gak ada alarm.

Pas gw cek log, baris ini:

```
Plugin 'telegram-platform' called register_platform() after its load timed out; ignored
WARNING gateway.run: No adapter available for telegram
```

Gatewaynya **berjalan**, tapi adapter Telegramnya tidak pernah kepasang. `systemd` melihat proses hidup → anggap sehat. Padahal bot di Telegram sudah mati.

## Kenapa bisa begitu

Root cause-nya rangkap dua:

1. Host ini punya route IPv6 yang mati. Koneksi ke Telegram API macet di AAAA record. Saat proses `systemd-resolved` baru saja boot dan DNS belum stabil, plugin Telegram mencoba terhubung, menunggu, dan akhirnya melebihi tenggat load-nya. Saat dia menyerah, adapter sudah tidak didaftarkan lagi.
2. Karena plugin load itu satu kali saat startup, kegagalannya bersifat permanen untuk sesi itu. Gak ada yang retry. Gatewaynya tetap `active`, tapi Telegramnya offline.

Kuncinya: **`systemctl` hanya memeriksa prosesnya, bukan fungsinya.** Sehat secara proses ≠ sehat secara fungsi.

## Fix-nya

Gw pasang watchdog: pengecekan berkala yang melihat apakah gateway benar-benar terhubung ke Telegram, bukan sekadar hidup. Indikatornya socket yang ter-`ESTABLISHED` ke Telegram API:

```bash
ss -tnp | grep 149.154.166.110:443
```

Ada socket = bot hidup. Gak ada = bot mati, walau `systemctl` bilang sehat.

Sebelumnya gw juga sudah pin IP Telegram di `/etc/hosts` biar DNS di awal boot gak macet:

```
# /etc/hosts
149.154.166.110 api.telegram.org
```

Lalu watchdog systemd yang jalan tiap 2 menit.

**Scriptnya** (`/usr/local/bin/hermes-tg-watchdog.sh`):

```bash
#!/usr/bin/env bash
# Pastikan adapter Telegram gateway Hermes selalu connect.
# Gateway bisa aktif (systemd happy) tapi adapter Telegram gagal load,
# jadi bot diam tanpa error service. Cek socket ESTAB ke Telegram API.
set -u

LOG=/var/log/hermes-tg-watchdog.log
TG_IP=${TG_IP:-149.154.166.110}   # api.telegram.org di-pin di /etc/hosts
MIN_GAP=${MIN_GAP:-300}           # jangan restart lebih sering dari ini (detik)

log() { printf '%s %s\n' "$(date -Is)" "$*" >> "$LOG"; }

# 1. gateway service harus aktif dulu
state=$(systemctl --user is-active hermes-gateway.service 2>/dev/null || true)
if [ "$state" != "active" ]; then
  log "gateway not active (state=$state) - starting"
  systemctl --user start hermes-gateway.service
  exit 0
fi

# 2. socket Telegram ada? kalau ada, sehat
if ss -tnp 2>/dev/null | grep -q "$TG_IP:443"; then
  exit 0
fi

# 3. gak ada socket -> restart, tapi throttle dulu
stamp=/run/hermes-tg-watchdog.last
last=0
if [ -f "$stamp" ]; then
  last=$(cat "$stamp" 2>/dev/null)
  case "$last" in ''|*[!0-9]*) last=0;; esac
fi
now=$(date +%s)
if [ $((now - last)) -lt "$MIN_GAP" ]; then
  log "telegram socket missing but restarted <${MIN_GAP}s ago - waiting"
  exit 0
fi

log "telegram socket missing - restarting gateway"
printf '%s' "$now" > "$stamp"
systemctl --user restart hermes-gateway.service
```

**Unit-nya** (`~/.config/systemd/user/hermes-tg-watchdog.service`):

```ini
[Unit]
Description=Hermes Telegram gateway watchdog
After=hermes-gateway.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/hermes-tg-watchdog.sh
```

**Timer-nya** (`~/.config/systemd/user/hermes-tg-watchdog.timer`):

```ini
[Unit]
Description=Run Hermes Telegram watchdog every 2 minutes

[Timer]
OnBootSec=10min
OnUnitActiveSec=2min
AccuracySec=30s

[Install]
WantedBy=timers.target
```

Pasang:

```bash
chmod +x /usr/local/bin/hermes-tg-watchdog.sh
systemctl --user daemon-reload
systemctl --user enable --now hermes-tg-watchdog.timer
```

## Kenapa cek socket, bukan ping Telegram API

Karena gw butuh tahu **botnya** terhubung, bukan server Telegramnya. Ping ke `api.telegram.org` cuma ngetes internet dan server Telegram — bot gw bisa tetap mati padahal internet baik-baik saja (persis kasus adapter gagal load). Socket yang ter-`ESTABLISHED` dari gateway ke Telegram API adalah satu-satunya indikator yang benar-benat menjamin **bot ini** sedang aktif polling.

Throttle 5 menit (`MIN_GAP=300`) biar gak restart storm: tiap restart nanggung plugin load ulang ±1 menit, dan kalau penyebabnya memang gagal total, restart berkali-kali dalam waktu singkat gak nambah apa-apa. Maksimal 12 restart/jam, sisanya menunggu.

## Hasil

Lampu padam terakhir: bot online lagi sendiri tanpa gw sentuh. Log watchdog bersih, gak ada penjelasan restart — pas lampu nyala, gateway restart sekali, socket tersambung, watchdog berhenti di langkah 2.

Sebelumnya: bot mati diam-diam, kadang sampai berhari-hari sampai gw sadar pesan gak dibalas. Sekarang maksimal 2 menit + 1 kali restart.

## Yang masih jadi batasan

- Watchdog ini mengamati **socket Telegram**, bukan mengatasi penyebab utamanya. Kalau saat boot DNS benar-benar tidak tersedia, gateway tetap mencoba restart sampai akhirnya bisa. Pin `/etc/hosts` sudah menangani sebagian besar skenario itu.
- `MIN_GAP=300` berarti saat gangguan sedang berlangsung, butuh waktu hingga 5 menit sebelum restart berikutnya. Cukup untuk kasus gw.
- Status `active` di `systemctl` masih tetap menyesatkan. Kalau kamu mengandalkan `systemctl` saja, kamu tidak akan tahu kalau botmu sedang mati. Ini pelajaran umum: proses yang sehat bukan berarti servis yang sehat.
