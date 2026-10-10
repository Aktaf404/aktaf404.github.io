---
title: Rumah 2 Lantai dari JSON — Saya Gambar Sendiri Pakai House Planner
date: 2026-10-09 23:30:00 +0700
categories: [Homelab, Desain]
tags: [docker, house-planner, self-hosted, denah, rumah]
image: /assets/img/house-planner/rumah-3d.png
---

Ini lanjutan dari post sebelumnya tentang House Planner. Sekarang rumahnya udah 2 lantai, dan yang mau aku ceritain adalah **gimana cara aku bikinnya** — bukan lewat editor gambar, tapi nulis JSON mentahnya langsung.

![Denah lantai 1](/assets/img/house-planner/denah-2d.png)
*Lantai 1 — ruang tamu, dapur open space, 2 kamar tidur, kamar mandi*

## Mulainya dari nol

Aku gak punya background arsitektur. Sopir B1 seharian, tapi waktu luang aku suka ngoprek komputer. Ide rumah udah lama ada, cuma gak pernah jadi gambar — selalu cuma bayangan.

Waktu lihat House Planner di GitHub, yang bikin aku tertarik itu formatnya terbuka: **satu rumah = satu file JSON**. Angka-angka di dalemnya millimeter asli, bukan satuan abstrak. Itu artinya aku bisa bikin denahnya dengan nulis teks, gak perlu belajar UI rumit.

## Cara aku bikinnya

**Langkah 1: Tentukan ukuran dasar.** Tanah 10×12 m di Jakarta Timur (realistis buat harga tanah sekarang). Bangunan 8×9 m, sisanya buat carport, taman belakang, taman samping. Satu lantai dulu waktu itu.

**Langkah 2: Nulis JSON.** Ini contoh dinding asli dari denahku:

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

Dinding depan, 8 meter panjang (dari x=1000 ke x=9000), tebal 150 mm (bata ringan), tinggi 2800. Titik `a` ke `b` adalah sumbu dindingnya. Bukaan — pintu, jendela — ditaruh di dinding dengan offset dari titik `a`.

**Langkah 3: Upload ke server** pakai script bawaan `plan-push.sh`. Buka browser, keluar denahnya. Selesai.

Satu hal yang keren: aku gak pernah klik-klik editor buat bikin denah utamanya. Semua dari nulis. Editor aku pake cuma buat lihat hasil dan ambil screenshot.

## Konsep ruang

Aku pikirin alurnya kayak orang beneran tinggal:

- **Pintu masuk** di depan, langsung ke ruang tamu + dapur yang open space — 36.8 m². Tamu gak perlu lewat lorong.
- **Dapur linear** di kanan: sink, kompor, kulkas sejajar, meja makan 4 kursi di ujung.
- **2 kamar tidur** di belakang, terpisah dari area umum biar bising dari ruang tamu gak masuk waktu tidur.
- **Kamar mandi** di tengah, masuk dari lorong kecil, bukan dari kamar manapun — supaya gak mengganggu privasi kamar.
- **Area cuci** terpisah di paling belakang, mesin cuci gak campur sama kamar mandi.
- **Carport** di depan 2.6×6 m, muat 1 mobil + sedikit ruang.
- **Taman** belakang buat bunga + samping buat jemur.

Total luas dalam **67.7 m²**, atau 74.6 m² kalau dihitung dinding.

## Naik ke 2 lantai

Minggu kemarin aku kepikiran: tanahnya udah beli, kenapa gak 2 lantai aja? Biaya bangunan naik, tapi luasnya bisa dobel tanpa nambah tanah.

Ada satu masalah teknis: **House Planner cuma support 1 lantai di 3D.** Di format plan-nya gak ada field "lantai 2" — dinding selalu mulai dari lantai dasar. Jadi gak bisa dibikin denah lantai 1 + 2 di satu file, terus lihat keduanya di 3D.

Jalan keluarnya: **1 lantai = 1 file**, disimpen jadi snapshot di server. App-nya punya versioning bawaan — setiap simpen bikin revisi, dan bisa bikin snapshot dengan nama. Jadi:

- Snapshot 1: **"Lantai 1 — 10×12 m, ruang tamu + dapur"**
- Snapshot 2: **"Lantai 2 — 2 kamar + ruang keluarga"**

Tinggal klik di menu Versions, denahnya switch instan. Footprint-nya sama persis (8×9 m), biar struktur dinding turunan aman.

### Lantai 2

![Denah lantai 2](/assets/img/house-planner/lantai2-denah.png)
*Lantai 2 — ruang keluarga, balkon, 2 kamar tidur, kamar mandi*

Perubahan ruting:

- **Ruang keluarga** gantin ruang tamu — lebih private, di lantai atas
- **Balkon** di depan, akses dari ruang keluarga lewat pintu geser 2.4 m. Buat ngopi sore.
- **Sudut baca** kecil di sisa ruang: bookcase, tanaman, sofa ambihan
- **Kamar utama** naik ke 2 lantai, kasur dilebarin dari 160 ke 180
- **Kamar mandi** dapet shower (lantai 1 cuma toilet + wastafel + mesin cuci)
- **Meja belajar** pindah ke kamar anak lantai 2, biar sepi waktu belajar

Yang aku jaga: tangga. Sayangnya **House Planner gak punya tangga di katalog furniturenya**, jadi aku kasih note aja di denah. Ini satu hal yang masih kurang dari app ini.

## Estimasi bahan

App-nya otomatis itung dinding, bukaan, material:

| Bagian | Estimasi |
|---|---|
| Pondasi batu kali | 12 m³ |
| Atap baja ringan | 90 m² |
| Lantai keramik 60×60 | 80 m² |
| Pintu + jendela alumunium | 10 unit (lantai 1), +6 lantai 2 |
| Instalasi (listrik, air, saniter) | 1 lot |

Harganya masih kasar, belum aku masukin harga toko bangunan terdekat. App-nya bisa diisi harga per material dan toko, cuma butuh waktu survei dulu.

## Yang aku pelajarin

Beberapa hal setelah nulis denah sendiri:

**Millimeter itu ternyata bikin pusing.** Dinding tebel 150 mm bukan 150 cm. Pertama aku tulis semua angka, baru sadar satuannya ribuan. Untungnya aku nulis JSON, jadi gampang koreksi — tinggal cari ganti.

**Furniture nggak sekadar tempat.** Ukuran asli katalog bikin aku sadar: kamar 3×3 m dengan kasur 160 + lemari 120 tebal beneran. Sebelnya jadi sempit. Akhirnya aku lebarin kamar ke 3.2 m, lemari pindah ke dinding lain.

**Bukaan itu strategi.** Jendela 1.4 m di dinding yang salah = panas sore masuk ke kamar. Aku pikirin arah matahari waktu naruh jendela — kamar tidur utama ngadep timur (pagi), ruang tamu ngadep utara (gak silau).

**Kamar mandi di tengah itu trik.** Awalnya mau masuk dari kamar utama (enak), tapi itu bikin kamar mandi jadi milik 1 kamar. Akhirnya masuk dari lorong — siapapun bisa akses tanpa masuk kamar orang.

## Versi dan backup

Salah satu fitur favoritku: **semua revisi disimpen di server.** Setiap kali denah berubah, file `versions/` di server dapet salinan dengan timestamp. Kalau aku salah naruh dinding, balik ke revisi sebelumnya.

![Tampilan 3D](/assets/img/house-planner/rumah-3d.png)
*View 3/4 lantai 1 — render WebGL di browser, server gak butuh GPU*

Kalau kamu pernah ngerjain file desain di Photoshop tanpa save-as versi, pasti tau rasanya fitur ini.

## Selanjutnya

- **Garis batas kavling asli** — ada fitur `site-fixed.json` yang bisa kunci koordinat tanah supaya gak sengaja geser
- **Harga real** — survei toko bangunan Jaktim, masukin harga per material
- **Tangga** — masih jadi lubang di denah, harus dimanualin
- **Listrik & saluran** — app-nya punya layer terpisah dengan trace otomatis + peringatan keamanan

## Buat yang mau coba

Source: [github.com/egmalt/house-planner](https://github.com/egmalt/house-planner) (MIT, gratis). Butuh Docker atau hosting PHP 8.1+, tanpa database, tanpa GPU.

Kalau kamu punya tanah dan kepikiran rumah, coba aja. Gak perlu jago gambar — aku buktiin, nulis angka millimeter di file teks pun bisa jadi rumah.
