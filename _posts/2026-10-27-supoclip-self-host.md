---
title: "SupoClip di Server 3 Core Tanpa GPU: Klip YouTube Otomatis, Tanpa Langganan OpusClip"
date: 2026-10-27 15:30:00 +0700
categories: [Homelab, Self-Hosting]
tags: [docker, supoclip, whisper, gemini, tailscale, video]
image: /assets/img/supoclip-clips.png
---

Kerjaan sampinganku sering harus potong video panjang jadi klip pendek buat Shorts, TikTok, Reels, dan Facebook. Cara manual: nonton ulang, cari momen lucu, potong, kasih subtitle, crop vertikal. Satu video satu jam bisa makan sejam lebih cuma buat itu.

OpusClip bisa semua itu otomatis, tapi berlangganan, ada watermark di paket murah, dan video-nya di-upload ke server orang. Aku nemu alternatif open source: **SupoClip** (`FujiwaraChoki/supoclip`, lisensi AGPL-3.0, sekitar 2 ribu star). Semua jalan di server sendiri, tanpa watermark, tanpa batas menit.

Yang bikin aku ragu awalnya: repo ini nyebut butuh API key AssemblyAI buat transkripsi (bayar) plus API key LLM. Server-ku cuma Intel i3-8100T 3 core, 15 GB RAM, tanpa GPU — bukan mesin video. Ternyata dua-duanya bisa digratiskan. Ini catatannya.

## Hasil dulu, biar gak penasaran

Video uji: klip musik 3 menit 32 detik dari YouTube. Waktu proses total **8 menit** dari tempel link sampai tiga klip siap download.

| Item | Hasil |
|---|---|
| Jumlah klip | 3 (otomatis dipilih AI) |
| Resolusi | 1080x1920 (9:16) |
| Durasi klip | 34s / 42s / 44s |
| Skor viral | 98 / 85 / 82 |
| Subtitle | kebakar, sinkron per kata |
| Biaya transkripsi | Rp0 (Whisper lokal) |
| Biaya LLM | Rp0 (Gemini free tier) |

Judul hook-nya juga ditulis AI dan dibakar di detik-detik awal klip: *"The ultimate commitment anthem"*, *"He knows the game and plays it"*, *"Why you are too shy to say it"*. Bahasa Inggris karena video sumbernya bahasa Inggris — subtitle dan pemilihan momen ikut bahasa aslinya.

![Tiga klip hasil generate, lengkap dengan skor viral](/assets/img/supoclip-clips.png)

Semua hasil generate ngumpul di halaman Library, lengkap dengan jumlah klip dan skor tertinggi biar gampang milih mana yang mau dipakai:

![Library SupoClip setelah satu video diproses](/assets/img/supoclip-library.png)

## Isinya apa

SupoClip itu pipeline lengkap, bukan sekadar pemotong:

- Tarik video dari YouTube, Twitch, Kick, atau upload file sendiri
- Transkripsi dengan timestamp per kata
- LLM memilih 3-7 momen paling berpotensi viral dari transkrip
- Skor hook, engagement, value, shareability untuk tiap klip
- Crop vertikal yang ngikutin wajah pembicara (face detection)
- Subtitle animasi, judul hook, opsional B-roll dari Pexels
- Editor bawaan: trim, split, merge, export preset TikTok/Reels/Shorts

Arsitekturnya enam container: frontend Next.js, backend FastAPI, worker ARQ buat antrean job, PostgreSQL, Redis, plus satu container MCP kalau mau dipakai dari Claude atau Cursor.

## Yang aku ganti biar gratis

Dua bagian berbayar di default, dua-duanya bisa diganti:

**Transkripsi.** Default-nya AssemblyAI (bayar per jam audio, tapi akurat dan cepat). Repo-nya ternyata punya jalur Whisper lokal yang tinggal diaktifkan satu variabel. Di server 3 core, model `base` selesai transkrip audio 3,5 menit dalam **sekitar 40 detik**. Jauh lebih cepat dari dugaanku — waktu 8 menit itu kebanyakan habis di render video, bukan transkripsi.

**Pemilih klip.** Butuh satu LLM. Aku pakai Gemini dari Google AI Studio yang ada free tier — request-nya cuma beberapa per video, jadi kuota gratisan lebih dari cukup. Model `gemini-3-flash-preview` sudah kelihatan bagus buat milih momen.

Efek sampingnya: transkripsi lokal artinya audio video orang **gak dikirim ke mana-mana**. Cuma transkrip teks yang dikirim ke Gemini.

## Cara pasangnya

Butuh Docker dan Git:

```bash
git clone https://github.com/FujiwaraChoki/supoclip.git
cd supoclip
cp .env.example .env
```

Isi `.env` minimal segini:

```env
TRANSCRIPTION_PROVIDER=whisper
WHISPER_MODEL_SIZE=base
LLM=google-gla:gemini-3-flash-preview
GOOGLE_API_KEY=isi_punyamu
BETTER_AUTH_SECRET=ganti_dengan_random
SELF_HOST=true
BACKEND_CPU_ONLY=true
FRONTEND_PORT=3107
```

`BACKEND_CPU_ONLY=true` penting kalau gak ada GPU — tanpa itu image-nya narik build CUDA yang ukurannya gila-gilaan. Secret auth bisa dibikin dengan `openssl rand -base64 32`.

Lalu nyalakan:

```bash
docker compose up -d --build
```

Build pertama lama: sekitar 25 menit di 3 core, karena isinya ffmpeg penuh plus torch CPU. Hasil akhirnya image backend dan worker masing-masing **4,5 GB**, frontend 2,3 GB. RAM idle setelah semua nyala sekitar 3 GB.

## Satu jebakan yang bikin gagal

Task pertamaku langsung error di tahap analisis transkrip:

```
Selected LLM provider is Google, but GOOGLE_API_KEY is not set
```

Padahal key-nya jelas ada di `.env`. Penyebabnya: aku ngisi key **setelah** `docker compose up` pertama jalan. Compose baca `.env` waktu container *dibuat*, bukan waktu container *start* — jadi container-nya masih megang nilai lama yang kosong.

Obatnya bukan rebuild, cukup recreate:

```bash
docker compose up -d
docker compose exec worker env | grep GOOGLE_API_KEY
```

Cek pakai `env` di dalam container, jangan percaya isi `.env` doang. Task yang sudah gagal bisa dilanjut tanpa ulang dari nol lewat tombol resume.

## Batasannya

Angka default yang perlu diketahui sebelum berharap banyak:

| Setting | Default | Arti |
|---|---|---|
| `MAX_VIDEO_DURATION` | 5400 | Video sumber maksimal 90 menit |
| `MAX_CLIPS` | 10 | Maksimal 10 klip per video |
| `clip_length: auto` | 15-60s | Target 25-50 detik |
| `clip_length: medium` | 30-60s | Paling aman buat TikTok/Reels |
| `clip_length: long` | 60-120s | Masih masuk Shorts (maks 3 menit) |

Batas platform yang aku cek hari ini: Shorts 3 menit, Reels 3 menit, TikTok upload sampai 60 menit, FB Reels 3 menit. Jadi klip 30-60 detik aman dipakai di keempatnya tanpa dipotong lagi.

Estimasi waktu: 3,5 menit video = 8 menit proses. Video 90 menit penuh kira-kira 1,5-2 jam. Kalau butuh lebih cepat, itu saatnya bayar AssemblyAI.

## Akses dan keamanan

Aplikasinya default bind ke `127.0.0.1`, jadi cuma bisa dibuka dari server itu sendiri. Aku buka lewat `tailscale serve` — HTTPS, tapi cuma untuk device yang login di tailnet-ku. Bukan funnel publik, gak ada yang bisa akses dari internet.

Satu hal yang perlu dicatat: bikin API key di dalam app **butuh email terverifikasi**, dan verifikasi butuh layanan email (SES). Buat pakai lewat browser gak masalah; buat automasi lewat API harus siapin itu dulu.

## Yang aku lewati

- **AssemblyAI** — diganti Whisper lokal, gratis tapi lebih lambat
- **Pexels B-roll** — butuh API key gratis, belum perlu
- **Apify** — fallback downloader berbayar buat Twitch/Kick, gak dipakai
- **Stripe dan billing** — mode monetisasi, jelas gak perlu buat pemakaian sendiri
- **Analytics (DataFast/PostHog)** — self-host gak ngirim apa-apa secara default, aku biarkan mati

## Worth it?

Buat yang rutin motong video panjang, ini jelas lebih hemat daripada langganan. Yang aku dapat: pipeline lengkap, tanpa watermark, tanpa batas jumlah video, dan audio gak keluar dari server. Yang aku bayar: waktu build pertama dan listrik server yang nyala terus.

Yang belum aku coba: upload file lokal berukuran besar, fitur publish langsung ke TikTok/Instagram (butuh OAuth tiap platform), dan B-roll. Kalau nanti sudah dicoba, bakal aku tulis lagi di sini.
