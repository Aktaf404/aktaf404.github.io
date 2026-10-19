---
title: "Repo APK Reverse Hilang dari GitHub, Tapi Skill-nya Masih Bisa Kuselamatkan"
date: 2026-10-19 19:30:00 +0700
categories: [Homelab, Security]
tags: [apk, reverse-engineering, github, skill]
---

Beberapa hari lalu aku nemu repo keren: **newliver666/apk-reverse**. Isinya bukan tutorial, tapi *skill* buat agent AI — semacam buku panduan plus skrip yang dibaca agent sambil kerja. Gunanya buat bongkar APK Android: hapus iklan, buang bloatware, patch dex, repack, re-sign, sampai analisis runtime.

Umurnya baru sebulan (dibuat 19 September 2026), tapi sudah 4.800-an star. License MIT. Wajar ramai — dokumentasinya rapi banget, ada indeks gejala, daftar jebakan, dan skrip yang siap dipanggil.

## Masalahnya: repo-nya sudah tidak ada

Waktu aku mau clone, hasilnya:

```text
$ curl -o /dev/null -w '%{http_code}' https://github.com/newliver666/apk-reverse
404
```

Bukan cuma repo-nya — akun `newliver666` juga ikut hilang:

```text
$ curl -o /dev/null -w '%{http_code}' https://github.com/newliver666
404
```

Jadi ini bukan repo yang di-private, bukan juga rename. Akunnya lenyap dari GitHub. Penyebabnya aku tidak tahu, dan aku tidak mau menebak.

Efeknya: semua tautan yang beredar — postingan, daftar trend, artikel — sekarang menunjuk ke halaman 404. Ibarat buku yang masih dikutip banyak orang tapi raknya sudah dikosongkan.

## Yang aku lakukan: cari cermin, bukan fork

Fork di GitHub mustahil, karena yang mau difork sudah tidak ada. Jadi jalannya cuma satu: cari salinan yang masih hidup.

Dua tempat yang masih menyimpan isinya:

- `git.nite07.com/flying1861/apk-reverse` — cermin git lengkap
- `skills.sh/newliver666/apk-reverse` — halaman katalog, isinya cuma ringkasan

Aku ambil dari cermin git-nya, karena itu satu-satunya yang membawa riwayat commit utuh:

```bash
git clone https://git.nite07.com/flying1861/apk-reverse.git apk-reverse
cd apk-reverse
git log --oneline | wc -l
# 61
git log -1 --format='%h %ad %s'
# 1b94651 Fri Oct 16 19:29:14 2026 +0800 Fix the bare path that turned CI red
```

61 commit, commit terakhir 16 Oktober 2026. Jadi salinan ini bukan sisa lama — dia ikut update sampai beberapa hari sebelum repo induknya hilang.

Setelah itu aku push ke repo sendiri, **private**, biar ada tempat yang jelas pemiliknya dan tidak menambah kebingungan di mesin pencari. Aku sengaja tidak bikin publik: yang punya hak adalah penulis aslinya, dan aku tidak tahu kenapa repo itu ditarik. Kalau nanti muncul lagi, tinggal sinkron ulang.

## Coba dipakai: bongkar APK F-Droid

Skill tanpa bukti cuma janji. Jadi aku uji pakai APK yang aman: **F-Droid**, toko aplikasi open source. Aku bongkar APK-nya sendiri, bukan aplikasi orang lain.

Pertama, cek dulu mesin ini sanggup apa:

```bash
python scripts/doctor.py --json
```

Awalnya statusnya `blocked`: belum ada Java, `apktool`, `apksigner`, `zipalign`, `adb`, `frida`. Setelah dipasang:

| Alat | Versi | Buat apa |
|---|---|---|
| openjdk | 21.0.12 | jalanin apktool & apksigner |
| apktool | 2.7.0 | bongkar & rakit ulang APK |
| zipalign / apksigner | build-tools 29 | rapikan & tanda tangan APK |
| adb | 34.0.4 | bicara ke HP |
| frida | 17.24.3 | hook runtime |
| capstone | 5.x | baca instruksi native |

Lalu bongkar APK-nya:

```bash
apktool d -f -s -o fdroid f-droid.apk
```

Flag `-s` bikin dex-nya dibiarkan utuh (tidak diubah jadi smali), jadi hasilnya masih bisa dibaca alat lain.

Hasil recon:

- Paket: `org.fdroid.fdroid`, versi 2.0.2, minSdk 24
- Empat arsitektur: arm64-v8a, armeabi-v7a, x86, x86_64
- Tiga dex: 8,9 MB + 9,2 MB + 1,6 MB (total 19,7 MB)
- 3.900 class milik `Lorg/fdroid/` kebaca tanpa decompiler
- Endpoint yang muncul: `f-droid.org`, forum, halaman donasi, schema Android

Terakhir, cek apakah ada SDK iklan nyempil:

```bash
python3 scripts/dex_strings.py fdroid --find 'admob|applovin|unityads|ironsource|mopub'
```

Nol hasil. Masuk akal — F-Droid memang tidak punya iklan.

## Yang belum bisa aku buktikan

Ini bagian yang sering dilewatkan orang waktu memamerkan tool baru, jadi aku tulis terang-terangan.

Mesin ini **tidak punya HP atau emulator** yang tersambung. Artinya:

- Bongkar APK: jalan
- Baca dex, cari referensi method, patch byte: jalan
- Repack + sign: jalan
- **Install dan buktikan patch-nya bekerja: belum**

Dan menurut skill-nya sendiri, hasil yang belum di-install dan dijalankan itu **prototipe**, bukan hasil jadi. Jadi aku tidak akan bilang "sudah berhasil menghapus iklan" untuk aplikasi apa pun. Yang sudah terbukti cuma satu: alur bacanya berfungsi di APK nyata.

Sisa yang kurang di mesin ini juga masih ada: `rasc`, alat bantu indeks dex yang harus di-build sendiri pakai Rust, dan untuk analisis dinamis butuh HP yang di-root plus `frida-server`.

## Catatan pemakaian

Tool seperti ini gampang disalahgunakan, jadi ini bukan sekadar formalitas:

- Pakai untuk APK milik sendiri, aplikasi open source, atau target yang memang kamu berhak uji
- Untuk belajar, F-Droid atau aplikasi open source lain sudah cukup
- Kalau aplikasinya bukan punyamu, izin dulu — bukan setelah ketahuan

Yang bikin repo ini layak diselamatkan bukan trik patch-nya, tapi disiplin di dokumentasinya: bedakan `observed` (sudah dijalankan), `inferred` (kesimpulan), `unverified` (kata orang), dan `unknown` (belum ada bukti). Buatku itu pengingat yang sehat — lebih baik bilang "belum tahu" daripada terdengar meyakinkan tanpa bukti.

Kalau kamu sempat menyimpan repo ini juga, cek hash commit terakhirnya. Kalau sama dengan `1b94651`, kita memegang salinan yang sama.
