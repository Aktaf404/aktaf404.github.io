---
title: "Remotion: Bikin Video Intro Portfolio Pakai React Code — di Mini PC Tanpa GPU"
date: 2026-10-15 09:00:00 +0700
categories: [Homelab, AI]
tags: [remotion, react, video, ffmpeg, mini-pc]
---

Lanjutan cerita oprek. Kali ini aku nyoba [Remotion](https://github.com/remotion-dev/remotion) — library yang bikin video pakai React code. Idenya: video = komponen React, tiap frame dirender browser, terus digabung jadi MP4. Buat yang biasa nulis frontend, ini rasanya kayak nemu jalan pintas.

## Kenapa tertarik

Aku butuh intro video buat portfolio. Biasanya pilihan cuma dua: pakai template CapCut (hasilnya keliatan template), atau belajar After Effects (nggak akan selesai). Remotion nawarin jalan ketiga: animasinya *dikutip*, bisa di-version control, dan bisa diganti teksnya dengan edit satu file.

## Yang pertama dicek: butuh GPU gak?

Mini PC aku 3 core, Intel UHD 630, tanpa GPU diskrit. Jawabannya: **gak butuh**. Remotion render pakai Chrome headless (software rendering via SwiftShader) + FFmpeg yang udah bundel. Jadi yang dibutuhkan cuma CPU dan kesabaran.

Perkiraan waktunya: video 2 detik @ 30fps = 60 frame, render ±30 detik di 3 core. Video 8 detik yang bakal aku tunjukin di bawah selesai render dalam ±1 menit.

## Instalasi

```bash
npx create-video@latest
```

Scaffold-nya interaktif (nanya install agent skills atau enggak), jadi kalau jalanin di server tanpa monitor, siap-siap jawab prompt-nya. Aku pilih template blank + skill `remotion-best-practices`.

Bonus yang kepikiran belakangan: skill-nya **lumayan bagus**. Ada guidance cara nulis animasi yang bener (pakai `interpolate()`, bukan CSS transition — karena CSS transition gak akan ke-render dengan benar di frame-based rendering), cara susun composition, dan layout rules buat video. Aku baca 4 file reference-nya sebelum nulis kode, dan itu ngefek — gak ada sekali pun aku ketemu error render karena pola animasi yang salah.

## Komponen videonya

Struktur kode intro-ku:

```tsx
const LetterName: React.FC = () => {
  const frame = useCurrentFrame();
  const { fps } = useVideoConfig();

  return (
    <div style={{ display: "flex", whiteSpace: "pre" }}>
      {NAME.split("").map((ch, i) => {
        const delay = 0.55 + i * 0.05;
        const t = interpolate(frame, [delay * fps, (delay + 0.7) * fps], [0, 1], {
          extrapolateLeft: "clamp",
          extrapolateRight: "clamp",
          easing: Easing.bezier(0.16, 1, 0.3, 1),
        });
        return (
          <span key={i} style={{
            opacity: t,
            translateY: `${interpolate(t, [0, 1], [46, 0])}px`,
            filter: `blur(${interpolate(t, [0, 1], [10, 0])}px)`,
          }}>{ch}</span>
        );
      })}
    </div>
  );
};
```

Tiap huruf masuk sendiri-sendiri dengan delay 0.05 detik — dari blur jadi tajam, turun dari atas. Ini yang di After Effects namanya stagger, tapi di sini cuma `.map()` + `interpolate()`.

## Animasi yang aku pake di V2

Setelah versi pertama (polos, cuma fade-in), aku bikin V2 yang lebih hidup:

- **Per-letter stagger** di nama, blur → sharp
- **42 partikel** drift + twinkle kayak langit malam — posisinya deterministic (pseudo-random seeded), jadi tiap render hasilnya identik
- **Light sweep** — pita cahaya tipis nyapuri background
- **Glow pulse** di accent line sekali pas selesai menggambar
- **Dua chip** yang spring-pop masuk ("Jakarta, Indonesia" + "Self-taught Builder")
- **Cinematic zoom-out** pelan dari 1.045× ke 1× selama 8 detik + fade to black di akhir biar clean buat loop

## Hasil

Video 8 detik, 1920×1080, h264, 789KB:

[Intro V2](/assets/files/portfolio-intro-v2.mp4) — yang ini final

[Intro V1](/assets/files/portfolio-intro-v1.mp4) — versi polos, buat pembanding

## Perbandingan dua versi

| | V1 | V2 |
|---|---|---|
| Durasi | 6s | 8s |
| Animasi teks | fade-in blok | per-letter stagger + blur |
| Background | flat hitam | partikel + glow + sweep |
| Ukuran file | 84KB | 789KB |
| Rasa | minimal banget | modern, ada depth |

## Catatan yang mungkin berguna

- **`translateY` gak ada di CSS properties Remotion** — yang valid itu `translate` dengan nilai `"0px 46px"` (X dan Y). Ini yang bikin TypeScript-ku nyandera 4x sebelum sadar. Error message-nya jelas kok, cuma perlu dibaca.
- **Frame-based, bukan time-based**: semua animasi pakai `useCurrentFrame()` + `useVideoConfig()` buat konversi detik → frame. Jangan hardcode angka frame.
- **Render = 0 API call.** Semua proses lokal. Yang makan waktu cuma baca reference skill, sisanya murni CPU.
- **Lisensi**: Remotion gratis buat individu/tim ≤3 orang, perusahaan lebih besar perlu lisensi komersial. Cek [remotion.dev/license](https://remotion.dev/license).

## Kesimpulan

Buat yang bisa React dan punya kebutuhan video yang *sama tapi beda-beda* (intro, outro, bumpers, video dari data), Remotion itu worth it. Yang paling kusukai: videonya ada di Git. Kalau klien minta ganti teks, aku edit satu baris, push, render ulang — bukan buka project AE 2GB.
