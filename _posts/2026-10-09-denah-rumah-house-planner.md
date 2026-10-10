---
title: Bikin Denah Rumah Sendiri pakai House Planner (Self-Hosted, Tanpa GPU)
date: 2026-10-09 23:30:00 +0700
categories: [Homelab, Desain]
tags: [docker, house-planner, self-hosted, denah, rumah]
image: /assets/img/house-planner/rumah-3d.png
---

Waktu luang di antara jadwal nyetir, aku iseng gambar denah rumah idaman. Bukannya beli software mahal, aku pasang **House Planner** — app open-source (MIT) yang self-hosted di mini PC.

Hasilnya: denah 2D skala asli, preview 3D, sampai estimasi bahan dan harga. Semua disimpen di server sendiri.

![Denah 2D rumah 10×12 m](/assets/img/house-planner/denah-2d.png)
*Denah 2D — skala asli dalam milimeter*

## Apa itu House Planner

Repo: [github.com/egmalt/house-planner](https://github.com/egmalt/house-planner)

App gambar denah rumah self-hosted. Fitur yang aku pakai:

- **Editor 2D skala asli (mm)** — dinding, pintu, jendela, furniture ukuran asli dari katalog
- **Preview 3D** — lihat rumah dari segala sudut
- **Layer utilitas** — saluran, listrik, air, pemanas lantai, lengkap dengan perhitungan
- **Bill of materials** — otomatis itung bahan + harga
- **Overlay satelit** — taruh denah di peta

Stack: **React + PHP, tanpa database**. Plan disimpen sebagai satu file JSON.

## Butuh GPU nggak?

**Nggak.** Ini pertanyaan pertamaku waktu lihat repo ini. Nama pembuatnya (homedesignsai.pro) bikin dugaan ada AI di baliknya — ternyata nggak. Yang ada:

- Server cuma nyimpen JSON — CPU biasa cukup
- Render 3D jalan di browser (WebGL), alias GPU di komputer/HP yang buka — ringan
- Nggak ada inferensi AI server-side

VPS paling murah atau shared hosting PHP udah cukup.

## Pasang

Docker, dua perintah:

```shell
git clone https://github.com/egmalt/house-planner.git
cd house-planner
docker compose up -d --build
```

Build-nya butuh beberapa menit (npm install di dalam image). Selesai, buka `/api/setup.php`, set password sekali, simpen, dan langsung jalan.

Aku naruh di mini PC lewat Tailscale, jadi bisa dibuka dari HP di mana aja aman.

## Desainnya

Tanah **10×12 m** di Jakarta Timur, bangunan 8×9 m, satu lantai, gaya minimalis tropis:

- **Ruang tamu + dapur open space** (36.8 m²) — dapur linear, meja makan 4 kursi
- **Kamar tidur utama** — kasur 160, lemari, meja samping
- **Kamar tidur anak** — kasur single, meja belajar
- **Kamar mandi** — toilet, wastafel, mesin cuci
- **Area cuci** terpisah di belakang
- **Carport** di depan (2.6×6 m), taman belakang + taman samping

Total **67.7 m²** luas dalam (74.6 m² termasuk dinding).

![Tampilan 3D dari depan](/assets/img/house-planner/rumah-3d-depan.png)
*View depan — pintu masuk samping carport, jendela besar menghadap jalan*

![Tampilan 3D isometrik](/assets/img/house-planner/rumah-3d.png)
*View 3/4 — terlihat semua zona dalam satu lihatan*

## Detail teknis yang keren

Beberapa hal yang bikin app ini beda dari editor denah biasa:

**Semua angka dalam milimeter bulat.** Dinding 150 mm, jendela 1400×900, tangga — semuanya skala asli. Bisa dijadikan acuan nyata buat tukang.

**Format plan terbuka.** Satu file JSON, dan ini benar-benar bisa dibaca + diedit manual. Ini contohnya (potongan):

```json
{
  "id": "w-front",
  "a": { "x": 1000, "y": 10500 },
  "b": { "x": 9000, "y": 10500 },
  "thickness": 150,
  "height": 2800,
  "materialId": "hebel-150"
}
```

Aku bikin denahnya **bukan** lewat editor, tapi nulis JSON ini langsung lewat chat AI, lalu upload ke server pakai script bawaan `plan-push.sh`. Sekali jadi, buka di browser — keluar denahnya. Editor tetap dipakai untuk visualisasi & koreksi kecil.

**Versioning bawaan.** Setiap perubahan disimpen ke `versions/` di server, dengan nomor revisi. Kalau salah naruh, balik ke revisi sebelumnya. Ada juga snapshot bernama.

**Estimasi otomatis.** Material dinding dihitung dari panjang×tinggi dinding, dan angka di tabel ini ikut pas diganti material:

| Bagian | Estimasi |
|---|---|
| Pondasi batu kali | 12 m³ |
| Atap baja ringan | 90 m² |
| Lantai keramik 60×60 | 80 m² |
| Pintu + jendela | 10 unit |
| Instalasi (listrik, air, saniter) | 1 lot |

## Yang masih bisa dioprek

- **Listrik & saluran belum aku gambar** — editor-nya punya mode trace otomatis dari titik ke titik, plus peringatan kalau ada yang nggak aman (jarak ke wastafel, dan lain-lain)
- **Harga masih estimasi kasar** — katalog materialnya bisa diisi dengan harga toko bangunan terdekat
- **Belum ada garis batas kavling asli** — fitur `site-fixed.json` bisa kunci boundary supaya nggak sengaja geser
- **Lantai 2** — sekarang masih 1 lantai; konsep 2 lantai tinggal ditambahkan

## Total biaya infrastruktur

Nol. Semua jalan di mini PC yang udah ada:

- Mini PC (Proxmox container) — idle, CPU kecil
- Docker container 1 buat app
- Tailscale untuk akses dari luar
- Domain gratis dari GitHub Pages buat blog ini

Yang ada biayanya cuma bangunan aslinya nanti — itu juga masih rencana, jadi angka di atas angka-angkaan, bukan tagihan.

## Coba sendiri

Kalau mau lihat atau pasang sendiri:

- **Source:** [github.com/egmalt/house-planner](https://github.com/egmalt/house-planner)
- **License:** MIT — gratis, bebas modifikasi
- **Requirement:** Docker atau hosting PHP 8.1+ (dengan Apache), tanpa database

Yang menarik dari proyek seperti ini: ternyata butuh alat mahal atau GPU untuk bikin sesuatu yang berguna. Yang dibutuhkan cuma kemauan dan satu malam luang.
