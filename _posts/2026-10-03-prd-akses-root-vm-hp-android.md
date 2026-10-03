---
title: "PRD: Akses Root VM dari HP Android, Tanpa IP Publik"
date: 2026-10-03 09:30:00 +0700
categories: [Homelab, AI]
tags: [tailscale, termux, ssh, vm, hermes]
---

> Ini versi PRD (Product Requirements Document) dari tutorial yg gw ikutin buat akses VM AI agent dari HP. Disimpen sebagai catatan biar gak lupa, sekaligus jadi acuan kalo mau dipasang lagi.

## Kenapa butuh ini

VM buat jalanin AI agent itu **ephemeral** — alamatnya bisa berubah, bisa di-reset kapan saja. Sementara gw tiap hari kerja bawa HP doang. Mau kudu bisa SSH ke VM **dari mana saja**, tapi **gak mau buka port publik** (bahaya, dan router rumah gak bisa port-forward gampang).

Jawabannya: **reverse tunnel via Tailscale**.

## Arsitektur

```
HP Android (Termux + Termius)     VM (Debian 12)
   127.0.0.1:2222   <───────────   root@:2222
        ▲                reverse tunnel
        │                (autossh, via Tailscale)
   Termius connect
```

Intinya: **VM yg nyambung ke HP**, bukan sebaliknya. HP gak perlu dibuka portnya. Semua traffic lewat tailnet (subnet `100.x`), gak ada satu pun port ke internet.

## Syarat

**HP Android:**
- **Termux** — wajib dari **F-Droid**. Versi Play Store udah kadaluarsa & sering crash
- **Tailscale** — aplikasi resmi
- **Termius** — SSH client, support import `.pem`

**VM:**
- Linux (Debian/Ubuntu), akses `root`
- `openssh-server` + `autossh`
- Join tailnet yg sama dgn HP

## Setup (sekali aja)

### 1. Termux di HP

```bash
pkg update && pkg install openssh
sshd
passwd        # catat password ini — cuma dipakai 1x
whoami        # catat username, misal: u0_a572
```

Buka aplikasi **Tailscale** → login → **Connect**. Catat IP HP-nya (format `100.x.x.x`).

⚠️ **Dua-duanya harus aktif:** Termux jalan + Tailscale connect (ada ikon VPN di status bar). Salah satu mati = tunnel putus.

### 2. Setup di VM (lewat agent AI)

Kirim prompt ini ke agent yg jalan di VM, isi bagian dalam kurung:

```text
Gw mau akses root ke VM ini dari HP Android gw via Termux.
Setup di HP gw udah beres:
- Termux openssh-server, sshd jalan di port 9922
- Tailscale connect (IP HP: [ISI IP TAILSCALE])
- Username Termux: [ISI USERNAME]
- Password Termux: [ISI PASSWORD]
Tolong kerjain:
1. Install & konfigurasi openssh-server (port 22 & 2222)
2. Generate SSH keypair, kasih file .pem buat diimport ke Termius
3. Tambahkan public key ke /root/.ssh/authorized_keys
4. Bikin reverse SSH tunnel VM → HP via Tailscale
   forward 127.0.0.1:2222 (HP) -> 127.0.0.1:2222 (VM)
5. Pasang autossh biar tunnel reconnect otomatis
```

Nanti VM balas dgn file `.pem` + detail koneksi.

### 3. Termius

- Import file `.pem` ke Keychain
- **New Host**:
  - **Host:** `127.0.0.1` ⚠️ **bukan** IP Tailscale
  - **Port:** `2222`
  - **Username:** `root`
  - **Key:** pilih key yg baru diimport
- Connect → langsung login sebagai root

> Kenapa host-nya `127.0.0.1`? Tunnel denger di **localhost HP**, bukan interface Tailscale. IP `100.x` gak akan nyambung.

## Reconnect tiap hari

Kalo tunnel putus, tinggal 3 langkah (< 10 detik):

1. Buka Termux → ketik `sshd` → Enter
2. Buka Tailscale → **Connect**
3. Buka Termius → Connect

autossh di sisi VM bakal reconnect otomatis. Lu cuma perlu pastiin Termux + Tailscale hidup di HP.

## Ketahanan

| Kasus | Perilaku |
|---|---|
| Sinyal HP drop | autossh auto-reconnect |
| VM reboot | service systemd auto-start |
| **VM reset total** | SSH hilang. Ulangi langkah setup VM di atas |
| Termux di-swipe-kill | Tunnel putus sampai dibuka lagi |

Yang paling sering: **VM reset total** (ephemeral). Makanya simpen prompt setup di atas, tinggal copy-paste lagi kalo perlu.

## Keamanan

- **Key auth aja** — password Termux cuma dipakai sekali pas setup, abis itu tunnel pake key
- **Gak ada port publik** — SSH cuma denger di `127.0.0.1`
- **Tailscale ACL** — tailnet policy pastiin cuma device sendiri yg bisa akses
- ⚠️ **File `.pem` itu = password root.** Jangan share kemana-mana.

## Yang bisa diperbaiki (v1.1)

- Login pake user biasa + `sudo`, bukan `root` langsung
- Healthcheck otomatis: cron cek tunnel, notif Telegram kalo putus lama
- Backup `.pem` terenkripsi

## Metrik sukses

- Dari HP di 4G, SSH connect **< 5 detik**
- Tunnel gak putus 7 hari tanpa intervensi manual
- Zero upaya manual setelah setup awal

---

*PRD lengkap (format dokumen) disimpen di repo: `_docs/PRD-akses-root-vm-hp-android.md`*
*Ide asli dari tutorial komunitas Muse AI, disesuaikan buat homelab pribadi.*
