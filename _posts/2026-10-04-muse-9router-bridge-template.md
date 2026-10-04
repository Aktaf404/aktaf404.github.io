---
title: "Muse → 9Router Bridge: Template Stack Portabel v2.1"
date: 2026-10-04 09:00:00 +0700
categories: [Homelab, AI]
tags: [9router, muse, bridge, hermes, systemd, tailscale]
---

> Ini catatan proyek: bikin **bridge** antara Muse AI (agent) dan gateway 9Router gw sendiri, dikemas jadi **template installable** biar bisa dipasang ulang di VM manapun. File-nya bisa diunduh di bawah.

## Masalahnya

Muse AI (agent favorit gw) itu **agent cloud** — gak ada API publik, cuma bisa diakses lewat web UI. Otaknya (memori, kepribadian, semua konteks gw) ada di sana, tapi tiap mau nanya harus **buka browser, login, ketik manual**.

Makanya gw pengen: **chat ke Muse lewat command line / Telegram**, tanpa buka browser. Dan ini harus terus jalan walaupun VM-nya ke-reset.

## Solusinya: bridge

Karena Muse gak punya API publik, gw bikin **jembatan** sendiri: shim OpenAI-compatible di dalem VM Muse, yg nulis request ke queue, lalu dijawab oleh **scheduled worker Muse sendiri** (pakai prompt khusus). Dari luar, kelihatannya kayak model API biasa.

```
client --HTTP--> 9router:20128 --HTTP--> shim:20501
  shim tulis queue/<id>.json, long-poll (maks 280 dtk)
  scheduled worker (cron 30 dtk) jawab & tulis responses/<id>.json
  shim baca response --> 9router --> client
```

Hasilnya: `hermes --model kamu/agent "pertanyaan"` dijawab **oleh Muse sendiri** (dengan memori & kepribadiannya). Dari sisi Hermes, `kamu/agent` ini salah satu model biasa. Dari sisi Muse, request masuk kayak chat biasa — dia gak sadar dipanggil dari luar.

Bonus: **response time jadi cepet** karena gak ada UI browser di tengah — langsung HTTP → queue → worker.

## Isi paket (v2.1)

Semua dipasang **di dalem VM Muse target**, mandiri dari luar:

| Komponen | Fungsi |
|---|---|
| **9router** (v0.5.95) | Gateway model AI lokal di `localhost:20128` |
| **muse-bridge** | Shim OpenAI-compatible → queue → worker Muse. Model: `kamu/agent` |
| **Pollinations** | Provider gratis keyless: `pllns/gpt-oss`, `pllns/openai-fast` |
| **caveman** | Token-saver proxy: `9router → caveman:8788 → Pollinations` |
| **headroom** | Token Saver proxy (context optimizer) di `:8787` |
| **filebrowser** | File manager web ringan di `:8090` |
| **tailscale** | VPN privat, akses mesin dari mana saja |
| **hermes** (opt-in) | Telegram gateway → bot jawab sebagai Muse via `kamu/agent` |
| **health-alert** | Alert Telegram saat service down/pulih (dedup) |
| **systemd units** | Unit + `keep.sh` keeper + template `container-app@.service` |

Model `kamu/agent` **disengaja gak lewat caveman/Pollinations** — itu bukan model publik, tapi terusan ke otak Muse sendiri.

## Bagian sulitnya: VM ephemeral

Yang istimewa dari template ini: VM tempat Muse jalan **bisa di-reset sewaktu-waktu**. Semua file di `/etc/systemd/system` hilang. Makanya ada **cron keeper** tiap 5 menit:

```bash
bash ~/workspace/systemd/keep.sh
```

`keep.sh` akan:
- Pasang ulang unit systemd yg hilang/berubah
- Pastikan service aktif (`Restart=always` udah nangani crash biasa)
- Update env proxy kalau ada
- Jalankan health check + alert Telegram **hanya saat ada transisi** (down→up atau up→down), biar gak spam

## Yang portabel, yg tidak

Template ini **jujur soal batas**. Gak semua bisa diotomatisasi:

**100% portabel (file biasa):** shim.py (Python murni, no deps), systemd units, keep.sh, health-alert.sh, register-providers.py (SQL langsung ke sqlite), config template.

**Butuh langkah manual di Muse target:**
- **Scheduled worker tiap 30 detik** — tempel `scheduled-worker-prompt.md` sebagai instruction
- **Cron keeper** — jadwalin `keep.sh` tiap 5 menit
- **`tailscale up`** — login browser (identity-bound, gak bisa otomatis)
- **Bot token Telegram** — kredensial diisi manual

Gak ada kredensial yg ikut di paket. API key di-generate fresh per instalasi.

## Cara pasang

```bash
# 1. (opsional) sesuaikan
cp config.sh config.local.sh
nano config.local.sh

# 2. install
bash install.sh
```

Installer **idempoten** — aman dijalanin ulang kapan saja. Komponen opsional diatur di `config.sh` (`INSTALL_CAVEMAN`, `INSTALL_HEADROOM`, `INSTALL_FILEBROWSER`, `INSTALL_TAILSCALE`, `INSTALL_HERMES_GATEWAY`, `INSTALL_CONTAINER`).

Setelah install, ikuti langkah manual di README (scheduled worker + cron keeper di Muse target).

## Uji end-to-end

```bash
# service hidup?
systemctl is-active 9router muse-shim caveman headroom filebrowser tailscaled

# provider gratis (langsung ke Pollinations via caveman)
bash ~/workspace/ask-9router.sh "jawab singkat: halo"

# bridge ke Muse (butuh scheduled worker aktif!)
curl -s http://localhost:20128/v1/chat/completions \
  -H "Authorization: Bearer $(cat ~/workspace/9router-apikey.txt)" \
  -H "Content-Type: application/json" \
  -d '{"model":"kamu/agent","messages":[{"role":"user","content":"siapa kamu?"}]}'
```

## Yang gw pelajarin

- **Long-poll > websocket** buat bridge kayak gini. Simpel, gak butuh connection state, dan 280 detik cukup buat worker agent ngerjain
- **Idempotent installer** itu sakti. VM reset? tinggal `bash install.sh` lagi, gak takut double-install
- **Health alert harus dedup.** Pertama bikin alert tiap 5 menit ngebanjirin Telegram. Sekarang cuma notif pas transisi
- **Jangan simpen kredensial di template.** Pernah kepretensi, sekali itu cukup

---

**📦 Download:** [muse-stack-template-v2.1.tar.gz](/assets/files/muse-stack-template-v2.1.tar.gz) (43 KB)

*Lisensi: pribadi — bebas dipake & dimodifikasi buat keperluan sendiri. Tanpa jaminan; jalankan di mesin sendiri.*
