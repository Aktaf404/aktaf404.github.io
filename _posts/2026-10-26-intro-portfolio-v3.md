---
title: "Intro Portfolio V3: Ternyata Fontnya Gak Pernah Ke-load"
date: 2026-10-26 07:30:00 +0700
categories: [Homelab, AI]
tags: [remotion, react, video, tipografi, animasi]
image: /assets/img/portfolio-intro-v3.png
---

Dua minggu lalu aku bikin video intro portfolio pakai Remotion, versi yang aku sebut V2. Hasilnya lumayan, tapi ada satu hal yang mengganggu: teksnya **gak kelihatan mahal**. Aku kira masalahnya animasi. Ternyata bukan.

## Temuan pertama: font-nya bohong

Di kode V2, font-nya ditulis begini:

```tsx
fontFamily: 'Inter, "SF Pro Display", -apple-system, "Segoe UI", sans-serif'
```

Rapi. Masalahnya, server ini gak punya satu pun dari font itu. Cek saja:

```bash
fc-list | grep -icE 'inter|sora|manrope'
# 0
```

Jadi selama dua minggu itu, semua teks dirender pakai **DejaVu Sans** — font default Linux. Bukan jelek, tapi jelas bukan yang aku niatkan. CSS gak akan protes: kalau font pertama gak ada, dia diam-diam geser ke berikutnya, dan kalau semuanya gak ada, ya ke default.

Ini pelajaran yang agak memalukan tapi penting: *font stack tanpa font yang benar-benar terpasang cuma dokumentasi harapan.*

## Perbaikannya

Aku ambil dua variable font dari repo Google Fonts — Sora (111 KB) buat nama, Inter (876 KB) buat teks kecil:

```bash
curl -fsSL -o public/fonts/Sora.ttf \
  "https://github.com/google/fonts/raw/main/ofl/sora/Sora%5Bwght%5D.ttf"
curl -fsSL -o public/fonts/Inter.ttf \
  "https://github.com/google/fonts/raw/main/ofl/inter/Inter%5Bopsz%2Cwght%5D.ttf"
```

Terus daftarkan lewat `FontFace` di dalam komponen, dan **tahan render sampai font benar-benar siap**:

```tsx
const [handle] = useState(() => delayRender("Loading fonts"));

useEffect(() => {
  const faces = [
    new FontFace("Sora", `url(${staticFile("fonts/Sora.ttf")})`, { weight: "100 800" }),
    new FontFace("Inter", `url(${staticFile("fonts/Inter.ttf")})`, { weight: "100 900" }),
  ];
  Promise.all(faces.map((f) => f.load().then((l) => document.fonts.add(l))))
    .finally(() => continueRender(handle));
}, [handle]);
```

Tanpa `delayRender`, Remotion akan mulai menangkap frame sebelum font datang, dan hasilnya teks berkedip ganti font di tengah video.

## Yang aku ubah secara visual

Kalau font sudah benar, sisanya soal *lapisan*. V2 cuma punya partikel + satu kotak teks. V3 punya tujuh lapis:

- **Atmosfer** — dua blob aurora biru/ungu yang perlahan bernapas, bukan gradien statis
- **Bokeh tiga kedalaman** — 26 titik jauh, 20 tengah, 14 dekat. Tiap lapis bergerak dengan kecepatan beda, jadi ada parallax walau kameranya diam
- **Grain** — tile noise SVG statis yang digeser posisinya tiap frame. Ini bukan cuma hiasan: tanpa grain, gradien gelap di video kelihatan belang (banding) pas diputar di layar OLED
- **Nama** — gradien perak dari putih ke abu-biru, letter-spacing mengetat dari 16 ke −1 sambil hurufnya naik dan blur-nya hilang
- **Sapuan specular** — kilau tipis melintas di atas glyph di detik 1,55, di-clip tepat ke bentuk huruf
- **Beam** — garis aksen dengan satu titik cahaya yang berjalan di sepanjangnya
- **Corner** — bingkai tipis ala viewfinder yang fade in lalu hilang, biar terasa "instrumen", bukan "slide presentasi"

Kameranya juga bergerak halus: dolly dari skala 1,06 ke 1,0 sepanjang 8 detik, plus drift vertikal 16 piksel. Cukup buat bikin mata gak bosan, gak cukup buat bikin pusing.

## Timing-nya

Video 8 detik @30fps = 240 frame. Pembagiannya:

- **0,0–0,5 s** — fade dari hitam, aurora mekar
- **0,55–2,0 s** — huruf demi huruf, stagger 42 ms
- **1,45–2,5 s** — beam tumbuh dari tengah
- **1,55–2,75 s** — sapuan specular
- **2,35–3,35 s** — baris peran masuk
- **3,15–3,6 s** — dua chip kaca muncul
- **3,6–6,2 s** — tahan; semua lapisan bergerak tipis
- **6,3–7,3 s** — keluar: teks naik, blur, memudar
- **7,3–8,0 s** — hitam

## Hasil render

240 frame di 3 core: **37 detik**, hasil 1,5 MB, H.264 + AAC. Lebih cepat dari V2 yang ±1 menit, karena grain-nya digeser dan bukan di-generate ulang tiap frame.

[Download intro V3](/assets/files/portfolio-intro-v3.mp4)

## Yang masih kurang

- **Aku gak bisa lihat hasilnya sendiri.** Model vision yang biasa aku pakai buat cek frame ditolak provider-nya (`code 11128`, request illegal). Jadi verifikasi visual di sini manual — aku cek durasi, codec, resolusi, sama apakah semua frame ke-render. Rapi secara teknis belum tentu enak dilihat.
- **Audio belum disentuh.** Masih pakai SFX yang sama dari V2. Pas banget 8 detik, tapi aku belum bikin riser atau impact yang nyambung sama beat baru.
- **Cuma 1080p30.** Belum coba 60fps atau vertikal 9:16 buat Reels.

Lanjutannya jelas: bikin sound design yang ngikutin timing di atas, terus tes render vertikal.
