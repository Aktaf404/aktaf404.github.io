# PRD — Akses Root VM dari HP Android (Tailscale + Termux + Termius)

**Versi:** 1.0
**Status:** Draft
**Penulis:** Homelab Malam
**Repo terkait:** Hermuse (Hermes + 9Router + Telegram stack)

---

## 1. Latar belakang

VM AI agent (Muse/Herombuse stack) bersifat **ephemeral** — bisa di-reset atau diganti
alamat publiknya kapan saja. Administrator sering tidak di depan komputer, hanya
membawa HP Android. Kebutuhan: akses SSH root ke VM **dari mana saja**, tanpa
membuka port publik, tanpa bergantung pada DNS/IP statis.

## 2. Tujuan

- Login SSH sebagai `root` ke VM dari HP Android, hanya lewat jaringan privat Tailscale.
- Tanpa IP publik, tanpa port forwarding di router, tanpa exposed attack surface.
- Tunnel pulih sendiri (autossh) dan mudah dihidupkan ulang dari HP.

## 3. Target pengguna

| Pengguna | Kebutuhan |
|---|---|
| Homelab operator (satu orang) | Debug VM dari HP saat bepergian |
| Multi-anggota tim kecil | Akses tunnel privat, bukan share password |

## 4. Arsitektur

```
HP Android (Termux + Termius)     VM Muse (deb12)
   127.0.0.1:2222   <─────────────   root@:2222
        ▲              reverse tunnel
        │              (autossh, via Tailscale)
   Termius connect
```

- HP dan VM keduanya join tailnet yang sama (Tailscale, subnet `100.x.x.x`).
- VM menjalankan reverse SSH tunnel ke HP: forward `127.0.0.1:2222 (HP)` → `127.0.0.1:2222 (VM)`.
- Termius di HP konek ke `127.0.0.1:2222`, auth sebagai `root` pakai key `.pem`.
- **Tidak ada port yang dibuka ke internet.** Semua lewat WireGuard tailnet.

## 5. Persyaratan

### 5.1 HP Android
- Termux dari **F-Droid** (versi Play Store sudah kadaluarsa, sering crash).
- Aplikasi Tailscale (Play Store).
- Termius (SSH client, support import key `.pem`).

### 5.2 VM
- OS Linux (Debian/Ubuntu), user `root` aktif, SSH key auth.
- Terinstall: `openssh-server`, `autossh`.
- Terhubung ke tailnet yang sama dengan HP.

## 6. Alur penggunaan

### 6.1 Setup awal (sekali di HP)

```
pkg update && pkg install openssh
sshd
passwd        # catat password, cuma dipakai sekali
whoami        # catat username, mis. u0_a572
```

Buka aplikasi Tailscale → login → Connect. Catat IP HP (format `100.x.x.x`).

### 6.2 Setup awal (di VM — berikan prompt ke agent AI)

```
Gw mau akses root ke VM ini dari HP Android gw via Termux.
Setup di HP gw udah beres:
- Termux openssh-server, sshd jalan di port 9922
- Tailscale connect (IP HP: [ISI IP TAILSCALE])
- Username Termux: [ISI USERNAME]
- Password Termux: [ISI PASSWORD]
Tolong kerjain:
1. Install & konfigurasi openssh-server (port 22 & 2222)
2. Generate SSH keypair, kasih file .pem untuk diimport ke Termius
3. Tambahkan public key ke /root/.ssh/authorized_keys
4. Bikin reverse SSH tunnel VM → HP via Tailscale
   forward 127.0.0.1:2222 (HP) -> 127.0.0.1:2222 (VM)
5. Pasang autossh biar tunnel reconnect otomatis
```

### 6.3 Setup Termius (sekali)

- Import file `.pem` ke Keychain.
- New Host:
  - **Host:** `127.0.0.1` (bukan IP Tailscale — tunnel hanya dengar di localhost HP)
  - **Port:** `2222`
  - **Username:** `root`
  - **Key:** pilih key yang diimport.
- Connect → langsung login sebagai root.

### 6.4 Reconnect harian

1. Buka Termux → ketik `sshd` → Enter.
2. Buka Tailscale → Connect → tunggu ikon VPN muncul.
3. Buka Termius → Connect.

Total waktu: **< 10 detik**.

## 7. Ketahanan & edge cases

| Kasus | Perilaku |
|---|---|
| Koneksi HP drop | autossh di VM reconnect otomatis |
| VM reboot biasa | autossh service systemd auto-start |
| VM di-reset total (ephemeral) | SSH hilang. Ulangi langkah 6.2 ke agent. |
| Termux di-close paksa | Tunnel putus sampai Termux dibuka lagi |
| Tailscale HP dimatikan | Tunnel putus; nyala lagi saat connect |

## 8. Keamanan

- **Key auth saja** — password Termux cuma dipakai sekali saat setup, lalu tunnel pakai key.
- **Tailscale ACL** — pastikan tailnet policy hanya izinkan device sendiri.
- **Tidak ada port publik** — SSH tidak pernah dengar di 0.0.0.0 VM, cuma localhost.
- ⚠️ Jangan share file `.pem` ke siapapun. Itu = password root VM.
- ⚠️ Password Termux diketik dalam chat hanya ke agent tepercaya; setelah tunnel jalan, ganti passphrase key.

## 9. Manajemen performa

- **Baterai HP:** minimal. Termux idle + Tailscale VPN ringan, hanya ada traffic saat session aktif.
- **Termux boleh diminimize** (jangan di-swipe-kill).

## 10. Roadmap (v1.1)

- [ ] Login tanpa root user (user `deploy` + `sudo`), prinsip least privilege
- [ ] Healthcheck proaktif: cron cek tunnel, notif Telegram kalau putus >5 menit
- [ ] Multi-device: izinkan >1 HP (multiple reverse tunnel / port terpisah)
- [ ] Backup `.pem` terenkripsi ke penyimpanan cloud

## 11. Sukses metrik

- Dari HP dalam kondisi 4G/5G, SSH connect **< 5 detik**.
- Uptime tunnel > 99% selama 7 hari, tanpa intervensi manual.
- Zero upaya manual setelah setup awal (kecuali VM reset total).

---

*Konten ini diturunkan dari tutorial komunitas Muse AI (muse.ai) dan disesuaikan
dengan kebutuhan homelab pribadi.*
