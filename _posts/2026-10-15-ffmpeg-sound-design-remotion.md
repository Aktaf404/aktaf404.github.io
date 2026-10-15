---
title: "Nambahin Sound Design ke Video Intro Remotion — Tanpa File Audio Pihak Ketiga"
date: 2026-10-15 12:00:00 +0700
categories: [Homelab, AI]
tags: [remotion, ffmpeg, sound-design, audio]
---

Masih tentang intro video portfolio. Sebelumnya udah kukerjain animasinya pakai Remotion, sekarang tantangannya: **suara**. Video tanpa audio itu kayak makan gak pake garam — secara teknis jalan, tapi ada yang kurang.

Pertanyaannya: suaranya dari mana?

## Opsi yang kutimbang

Aku butuh tiga suara buat intro 8 detik: riser (build-up), whoosh (transisi), dan chime (penutup yang manem). Tiga jalur yang kupertimbangkan:

1. **Stable Audio Open** (AI generator open-source) — keren, tapi modelnya gated di HuggingFace, butuh install PyTorch ±2GB, dan di CPU 3 core aku satu clip bisa 5–15 menit. Terlalu mewah buat kebutuhan 3 suara pendek.
2. **Freesound.org** — kualitas pro, gratis, instan. Tapi: harus cari-cari yang pas, cek lisensi satu-satu, dan hasilnya "punya orang".
3. **Synthesize sendiri pakai FFmpeg** — yang udah terpasang di server, nol download, hasilnya 100% orisinal.

Kupilih nomor 3. Selain nol risiko lisensi, alasan paling jujur: penasaran apakah FFmpeg bisa bikin suara yang layak.

## Resep tiap suara

Semua dibuat dengan filter FFmpeg, tanpa satu pun file audio masuk:

**Riser** — sine sweep yang pitch-nya naik kuadratik (60Hz → ±220Hz dalam 2.2 detik), lowpass biar gak kasar, fade in/out:

```bash
ffmpeg -f lavfi -i "aevalsrc=0.28*sin(2*PI*(60*t+30*t*t)):s=44100" \
  -af "lowpass=f=500,afade=t=in:st=0:d=0.4,afade=t=out:st=1.6:d=0.6" \
  riser.wav
```

Kunci suara "halus" ada di rumus `60*t + 30*t²` — kenaikan pitch-nya makin cepet makin akhir, persis feel riser di trailer (yang murahan malah linear, kedengeran kayak sirene).

**Whoosh** — pink noise (bukan white — pink lebih gelap dan natural), difilter biar tinggal range "angin", plus envelope naik-turun:

```bash
ffmpeg -f lavfi -i "anoisesrc=color=pink:amplitude=0.5" \
  -af "highpass=f=300,lowpass=f=3000,afade=t=in:st=0:d=0.35,afade=t=out:st=0.55:d=0.65,volume=0.35" \
  whoosh.wav
```

**Chime** — yang paling seru: tiga nada harmonik (E5 = 659.25Hz, B5 = 987.77Hz, E6 = 1318.5Hz) masing-masing dengan decay eksponensial, plus echo tipis:

```bash
ffmpeg -f lavfi -i "aevalsrc=0.18*sin(2*PI*659.25*t)*exp(-1.8*t)+0.12*sin(2*PI*987.77*t)*exp(-1.4*t)+0.06*sin(2*PI*1318.5*t)*exp(-2.2*t):s=44100" \
  -af "aecho=0.6:0.4:180:0.25,afade=t=out:st=2.2:d=0.8" \
  chime.wav
```

`exp(-1.8*t)` itu yang bikin suara "berdenting lalu memudar" — persis fisika nyaring bel/kalimba. Nada E5+B5 sengaja dipilih karena intervalnya consonant (kuint), jadi manem di kuping.

## Mixing: di sinilah versi pertama gagal

Rakit timeline pakai `adelay` + `amix`:

| Track | Mulai di | Volume |
|---|---|---|
| riser | 0.0s | 0.55 |
| whoosh | 1.5s (pas nama kelar masuk) | 0.5 |
| chime | 3.3s (pas aksen line + chip muncul) | 0.55 |

Percobaan pertama langsung kena masalah klasik: **whoosh dan chime keburu kecil**. Kuverifikasi pakai `volumedetect` per track:

```
riser:  mean -15.7 dB
whoosh: mean -39.0 dB   ← kecil banget
chime:  mean -38.5 dB   ← sama
```

23dB bedanya sama riser — di earphone bakal cuma kedengeran riser doang. Fix-nya: boost per-track (whoosh +10dB, chime +12dB), lalu taruh **compressor + limiter** di master biar yang keras gak clipping:

```bash
[0:a]volume=0.55,adelay=0|0[r];
[1:a]volume=3.2,adelay=1500|1500[w];
[2:a]volume=4.0,adelay=3300|3300[c];
[r][w][c]amix=inputs=3:duration=longest:dropout_transition=0,
apad=whole_dur=8,
acompressor=threshold=-18dB:ratio=3:attack=8:release=120,
alimiter=limit=0.85,
afade=t=out:st=7.3:d=0.7,
loudnorm=I=-16:TP=-2:LRA=9
```

`loudnorm=I=-16` buat menyamakan loudness ke standar web streaming.

## Verifikasi: jangan cuma denger, ukur

Hal yang kupelajari dari oprek sebelumnya: **jangan percaya "kayaknya udah bener"**. Kukuk level akhir per detik pakai volumedetect:

```
detik 1: -12.2 dB  (riser)
detik 2: -20.0 dB
detik 3: -14.1 dB  (whoosh + mulai chime)
detik 4: -17.0 dB  (chime)
detik 5: -29.8 dB  (decay)
```

Dan di file final render: mean -16.7 dB, peak -2.1 dB — cukup keras, nol clipping. Timing riser → whoosh → chime kelihatan jelas dari profil level-nya.

## Integrasikan ke Remotion

File final jadi `public/intro-sfx.wav`, terus di komponen tinggal dua baris:

```tsx
import { Audio } from "@remotion/media";
import { staticFile } from "remotion";

// di dalam AbsoluteFill:
<Audio src={staticFile("intro-sfx.wav")} premountFor={fps} />
```

Satu jebakan kecil: `staticFile` itu dari package `remotion`, **bukan** `@remotion/media` — TypeScript-ku nyandera sekali sebelum sadar.

Hasil render: MP4 8 detik dengan audio AAC, 1.1MB.

## Hasil akhir

[Download video intro dengan sound design](/assets/files/portfolio-intro-v2-sfx.mp4)

Hasil dengar: riser halus nyusul huruf-huruf masuk, whoosh pas nama komplet, chime manem bareng munculnya line aksen dan chip. Buat suara hasil sintesis murni FFmpeg, jujur lumayan banget.

## Kesimpulan

FFmpeg itu bukan cuma converter — `aevalsrc` + filter audio itu synth yang cukup layak buat sound design sederhana. Untuk kebutuhan SFX pendek yang repetitif (intro/outro/bumper), pendekatan ini lebih cepet daripada AI generator, lebih bersih daripada hunting sample, dan hasilnya ada di Git bersama kode videonya.

Kapan pakai AI generator (Stable Audio Open)? Kalau butuh suara yang *impossible* buat disintesis — tekstur ambience, suara benda nyata yang spesifik. Untuk whoosh dan chime? FFmpeg udah lebih dari cukup.
