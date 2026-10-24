---
title: "Punya Sendiri Perpustakaan Prompt: prompts.chat di Docker, Akses Cuma Lewat Tailnet"
date: 2026-10-24 09:00:00 +0700
categories: [Homelab, Self-Hosting]
tags: [docker, prompts-chat, tailscale, postgres, nextjs]
image: /assets/img/prompts-chat-library.png
---

Repo `f/prompts.chat` itu salah satu proyek prompt paling terkenal — dulu namanya *Awesome ChatGPT Prompts*, sekarang 143 ribu star. Isinya kumpulan prompt siap pakai buat ChatGPT, Claude, Gemini, dan sejenisnya, plus buku interaktif soal teknik prompting.

Yang bikin menarik: **semuanya bisa di-host sendiri.** Jadi aku pasang satu instans di server, dan perpustakaan prompt-nya jadi punya sendiri — bukan nyontek situs orang tiap kali butuh ide.

## Isinya apa aja

Aplikasinya Next.js, datanya di PostgreSQL. Ada kategori, tag, komentar, voting, versi prompt, dan koleksi pribadi. Prompt content-nya berlisensi CC0 (domain publik), kodenya MIT.

Bonus yang jarang disebut: di dalam repo ada buku **The Art of ChatGPT Prompting** — 28 bab, dari dasar sampai teknik kayak chain-of-thought, few-shot, dan agent. Semua ikut ke-host, jadi bisa dibaca offline dari instans sendiri.

## Cara pasangnya

Paling gampang lewat Docker Compose, karena image-nya udah jadi di `ghcr.io` — gak perlu build Next.js dari nol:

```bash
git clone https://github.com/f/prompts.chat.git
cd prompts.chat
```

Bikin `.env` isi secret autentikasi:

```bash
printf 'AUTH_SECRET=%s\nPORT=4444\n' "$(openssl rand -base64 32)" > .env
chmod 600 .env
```

Lalu nyalakan:

```bash
docker compose up -d --wait
```

Dua kontainer naik: `app` dan `db` (postgres:17). Cek kesehatannya:

```bash
curl -s http://localhost:4444/api/health
```

```json
{"status":"healthy","database":"connected"}
```

Database langsung keisi sekitar **1006 prompt** contoh, jadi gak mulai dari kosong.

## Angka nyatanya di server

- Image `app`: **4.5 GB** (sekali unduh, setelah itu cuma update)
- RAM terpakai saat jalan: **±1 GB**
- Waktu dari `up` sampai sehat: **±40 detik**
- Port aplikasi: `4444`

Server ini RAM-nya 16 GB, jadi satu kontainer segini kecil. Yang perlu diingat justru disk — image Next.js itu gendut.

## Kenapa gak aku buka ke internet

Awalnya aku sempat buka lewat Tailscale Funnel. Tiga menit kemudian aku tutup lagi, karena dua alasan yang cukup jelas begitu dipikir:

1. **Default-nya registrasi terbuka.** Siapa pun yang tahu URL bisa bikin akun di perpustakaan prompt pribadi. Bisa dimatikan, tapi tetap — ini situs yang isinya cuma buat aku sendiri.
2. **Gak ada gunanya publik.** Ini bukan produk yang mau dibagi ke orang banyak, cuma alat bantu nulis prompt.

Jadi sekarang jalurnya cuma **Tailscale Serve**, bukan Funnel:

```bash
tailscale serve --bg --https=10002 http://127.0.0.1:4444
```

Hasilnya `https://srv-01.tailb504f8.ts.net:10002` — HTTPS, sertifikat valid, tapi **hanya bisa dibuka perangkat yang login ke tailnet aku**. Dari luar, port-nya ya mati.

Registrasi juga aku matikan lewat override Compose biar gak ketimpa kalau `compose.yml` di-update:

```yaml
services:
  app:
    environment:
      PCHAT_ALLOW_REGISTRATION: "false"
```

Sudah dicek: percobaan daftar pakai email tes hasilnya ditolak, dan tabel user gak nambah.

## Yang gak aku pasang

Biar jujur, ini daftar yang sengaja dilewat:

- **Login GitHub/Google** — ribet bikin OAuth app buat satu pengguna. Login kredensial lokal cukup.
- **AI search & AI generation** — dua fitur ini butuh `OPENAI_API_KEY`, artinya bayar per pemakaian. Pencarian teks biasa sudah lebih dari cukup untuk 1000 prompt.
- **Reverse proxy Nginx** — Tailscale Serve sudah kasih HTTPS, gak perlu nambah lapisan lagi.
- **Backup otomatis** — belum. Padahal tinggal satu baris `pg_dump` di cron. Ini yang paling mungkin aku tambah berikutnya.

Total waktu pasang: di bawah 10 menit, dan sebagian besar habis buat nunggu image 4.5 GB keunduh.

Sumber: [github.com/f/prompts.chat](https://github.com/f/prompts.chat) · [panduan Docker](https://github.com/f/prompts.chat/blob/main/DOCKER.md)
