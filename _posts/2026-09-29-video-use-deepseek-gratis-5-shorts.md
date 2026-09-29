---
title: video-use + DeepSeek Gratis, 5 Shorts dari Live 2 Jam
date: 2026-09-29 21:00:00 +0700
categories: [Media, AI]
tags: [video-use, deepseek, ffmpeg, shorts]
---

Mau potong live stream 2 jam jadi 5 Shorts vertikal. OpusClip $23/bulan. Cara ini **Rp 0**.

## Bahannya

- **video-use** (github.com/browser-use/video-use) — skill open source buat agent edit video
- **Agent** — opencode, model `deepseek-v4-flash` (gratis lewat 9router)
- **ffmpeg** + **yt-dlp**

## Trik utamanya

Model murah **gak bisa** bikin dari nol. Tapi **jago ngikutin**. Jadi:

1. **Sekali aja**, clip satu live pake model pintar. Hasilnya folder `edit/` isinya:
   - `edl.json` — daftar Shorts (judul, detik mulai-akhir, layout, quote)
   - `project.md` — "memori" sesi: keputusan crop, grade, subtitle
   - `render.py` — script ffmpeg: potong, crop, fade, loudnorm, burn subtitle
   - `subs.py` — bikin ASS subtitle (2 kata/chunk, UPPERCASE) + tabel FIXES
2. **Live berikutnya**, suruh DeepSeek **baca folder itu dulu**, terus bikin versi baru. Rumusnya udah ada, dia tinggal ngisi.

## Subtitle

Jangan pake ElevenLabs. Auto-subs **YouTube** udah cukup:

```bash
yt-dlp --write-auto-subs --sub-format json3 <url>
```

Timestamp-nya akurat, teksnya kotor. Makanya ada `json3_to_scribe.py` — convert ke format Scribe. Terus tabel `FIXES` di `subs.py` buat benerin salah dengar (`jpbt → GPT`, `agen → agent`).

## Yang penting

- **Minta usulan topik dulu** sebelum render. Lu yg tau momen mana bagus, model gak
- Model murah **gak bisa liat gambar**. Layout/crop harus udah bener di folder contekan. Kalo setup OBS berubah, jalanin model pintar lagi sekali
- FIXES per kata. Salah dengar kepeceh 2 kata (`Chat` + `JPBT`) gak kena fix multi-kata

## Biaya

DeepSeek Flash lewat 9router = Rp 0. Sempet kepikir direct API DeepSeek ($0.06 per live, min top-up $2) tapi 9router udah punya model yg sama gratis. Gak jadi.

Total: 16 menit kerja, 5 Shorts, Rp 0.
