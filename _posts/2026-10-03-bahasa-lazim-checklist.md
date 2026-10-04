---
title: "Bahasa Lazim: Checklist Nulis Teks Indonesia Biar Bener dan Lazim"
date: 2026-10-03 20:00:00 +0700
categories: [Writing, AI]
tags: [bahasa-indonesia, ai, writing, checklist, kotakin]
---

> Ini rangkuman checklist **Bahasa Lazim** — aturan nulis teks bahasa Indonesia buat konten bikinan AI. Cocok buat: teks UI, konten web, data contoh, soal, atau apa pun yg dibaca user Indonesia. PDF aslinya bisa diunduh di bawah.

## Latar belakang

Logikanya bener, tesnya lulus, tapi tetep ada **kompor di teras**.

Itu masalahnya. Pas nge-generate konten pake AI, teksnya *secara tata bahasa* bener — tapi gak *lazim*. Gak kayak orang Indonesia ngomong. User baca, ngerasa aneh, tapi gak tau kenapa.

Konsepnya diambil dari **ASD-STE100** (Simplified Technical English — standar internasional buat nulis manual pesawat): aturan nulis bernomor + kamus kata yg boleh dan nggak boleh dipakai. Versi ini diadaptasi ke bahasa Indonesia, semua aturan dan contoh diambil dari koreksi asli pas bikin **100 soal detektif** (Kotakin) pake AI.

> Tes ngecek bener atau salah. Nggak ngecek lazim atau nggak.

## Bagian 1: Aturan

### 1. Kata

**1.1** Pakai kata yg dipakai orang sehari-hari, bukan kata kamus atau sastra.

- ✗ Tas titipan penumpang **raib**. → ✓ Tas titipan penumpang **hilang**.
- ✗ Rekaman belum **dicadangkan**. → ✓ Rekaman belum **disalin**.

**1.2** Kalau orang biasa pakai kata serapan atau kata gaul, pakai itu.

- ✗ Ponsel yg sedang **diisi daya**. → ✓ Ponsel yg sedang **dicas**.

**1.3** Hindari kata daerah kalau user lu dari seluruh Indonesia.

- ✗ **dingklik** → ✓ **bangku pendek**

**1.4** Pilih ejaan yg orang kenali, walaupun bukan yg paling baku.

- ✗ **jeriken** oli → ✓ **jerigen** oli

**1.5** Satu hal, satu kata, di semua tempat: judul, isi, pertanyaan, label.

Kalau di judul "dicas", di cerita juga "dicas" — bukan "diisi daya".

### 2. Nama (benda, tempat, fitur)

**2.1** Maksimal 3 kata. Buang keterangan yg nggak membedakan.

- ✗ donat pesanan kantor → ✓ **donat**
- ✗ mikrofon nirkabel → ✓ **mikrofon**
- ✗ durian berpita → ✓ **durian**

**2.2** Nama tempat harus ada di dunia nyata. Kalau nggak ada papan nama kayak gitu di dunia nyata, jangan pakai.

- ✗ Ruang Sortir, Ruang Dengar, Ruang Pajang, Selasar Parkir
- ✓ **Gudang, Kantor, Lapak, Parkiran**

**2.3** Jangan bikin nama dari terjemahan istilah asing.

- ✗ Ruang Kurator → ✓ Ruang Staf
- ✗ Pantri → ✓ **Dapur**

### 3. Kalimat

**3.1** Pertanyaan atau instruksi: maksimal 2 kalimat. Fakta dulu, baru tanya atau perintah.

- ✓ Ranselnya hilang. Siapa yg mengambilnya?

**3.2** Satu pertanyaan maksimal 13 kata.

**3.3** Jangan selipkan detail yg bikin orang bertanya-tanya.

- ✗ Piringan hitam **langka** belum sempat **dihitung**. ("langka" kenapa? "dihitung" apanya?)
- ✓ Kardus kiriman kolektor hilang.

**3.4** Kalimat harus bisa dimengerti tanpa baca teks sebelumnya.

- ✗ Kabelnya masih tergantung, ponselnya tidak ada.
- ✓ Ponsel pelanggan hilang saat dicas.

### 4. Isi dan dunia nyata

**4.1** Teks cuma boleh nyebut yg kelihatan.

- ✗ Kardus "tidak ada di meja kasir", padahal di denah nggak ada meja kasir.

**4.2** Nama harus sama dengan gambarnya.

- ✗ **sopir truk**, padahal gambarnya mobil. → ✓ **sopir mobil**
- ✗ Mobil di pangkalan ojek. → ✓ **Motor.**

**4.3** Isi harus masuk akal di tempatnya. Tulis aturannya eksplisit ("X cuma boleh di Y"), jangan berharap AI nebak.

- ✗ Kompor di teras, lemari obat di ruang cukur, TV di warung makan, barbel di ruang ganti, lemari arsip di dapur.

**4.4** Tempat kosong juga salah.

- ✗ Gudang kain tanpa rak kain. → ✓ Gudang kain diisi rak kain.

**4.5** Cek kenyataannya, bukan cuma bahasanya.

- ✗ **Pengendali kembang api**, padahal kembang api dinyalain pake api. → ✓ **Kotak kembang api**

### 5. Verifikasi

- **5.1** Setelah ganti kata apa pun, jalanin ulang tes. Ganti satu kata bisa ngerusak logika.
- **5.2** Baca pake mata user yg baru pertama kali lihat, bukan mata yg udah tau konteksnya.
- **5.3** Boleh nanya AI "ini janggal nggak?", tapi keputusan akhirnya dari lu.
- **5.4** Masukin aturan dan kamus ini ke file konteks project (AGENTS.md), biar nggak kejadian lagi.

## Hasilnya: angka mentah

Diterapin ke 110 soal Kotakin setelah dikoreksi:

| Yg dicek | Aturan | Hasil |
|---|---|---|
| Nama benda | maks 3 kata | 106 dari 110 nama ≤ 3 kata |
| Pertanyaan | maks 2 kalimat | 110 dari 110: fakta + "Siapa yg …?" |
| Pertanyaan | maks 13 kata | terpanjang 13 kata, median 9 |

## Bagian 2: Kamus

✓ = pakai. ✗ = jangan, ganti dgn kolom "Pakai".

| Jangan ✗ | Pakai ✓ | Contoh benar |
|---|---|---|
| raib | hilang | Tas titipan hilang. |
| diisi daya | dicas | Ponsel hilang saat dicas. |
| dicadangkan | disalin | Rekaman belum disalin. |
| pantri | dapur | Kue disimpan di dapur. |
| selasar parkir | parkiran | Parkiran |
| ruang pajang | lapak / toko | Lapak |
| ruang sortir | gudang | Gudang |
| ruang dengar | kantor | Kantor |
| ruang kurator | ruang staf | Ruang Staf |
| ruang kemas | ruang pengemasan | Ruang Pengemasan |
| ruang pamer | ruang pameran | Ruang Pameran |
| kotak rendang | paket kiriman | Paket kiriman berisi rendang |
| dingklik | bangku pendek | Bangku pendek |
| jeriken | jerigen | Jerigen oli |
| pengendali kembang api | kotak kembang api | Kotak kembang api hilang. |

## Kenapa ini penting buat yg pake AI

AI nulis cenderung "benar tapi tidak lazim". Model dilatih dari teks formal (berita, dokumen, wiki), jadi default-nya: *raib*, *dicadangkan*, *diisi daya*, *Ruang Kurator*. Semua bener secara kamus, semua aneh secara nyata.

Checklist ini jembatannya: aturan bernomor yg bisa lu tempelin ke prompt atau file konteks project, plus kamus kata konkret. Efeknya: konten jadi terbaca kayak ditulis orang, bukan terjemahan mesin.

---

**📄 Download:** [bahasa-lazim-checklist.pdf](/assets/files/bahasa-lazim-checklist.pdf) (230 KB, 4 halaman)

*Sumber: pixel.developer · contoh-contohnya dari koreksi asli waktu bikin Kotakin (kotakin.id). Konsep ASD-STE100 dipinjam (aturan bernomor + kamus), bukan STE resmi.*
