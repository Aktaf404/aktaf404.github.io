---
title: "Need for Speed Underground 2 Jalan Native di R36S, Bukan Emulator"
date: 2026-10-23 10:00:00 +0700
categories: [Gaming, Handheld]
tags: [r36s, rk3326, ports, nfsu2, arm64]
---

Handheld murah kayak R36S itu otaknya chip Rockchip RK3326 — ARM 64-bit, RAM 1 GB, GPU Mali-G31. Buat emulasi PlayStation 2 jelas tidak kuat. Tapi ada jalan lain: **recompiled port**, yaitu kode game PC lama diterjemahkan ulang jadi biner ARM64 native, bukan diemulasi.

Itu yang dikerjakan proyek **NFSU2 Recompiled**, dan sekarang sudah ada build khusus buat handheld RK3326.

## Apa yang dirilis

Repo `manuelpllull/nfsu2-recompiled-r36s`, tag `v1.0.0-r36s-20261017`, rilis 17 Oktober 2026. Isinya satu file `NFSU2-R36S.zip` (42,5 MB) yang sudah berisi biner jadi `nfsu2.aarch64` — jadi tidak perlu compile sendiri.

Targetnya: R36S, RG351MP, RGB20S. Sistem operasi ArkOS, AmberELEC, atau ROCKNIX.

Yang diperbaiki di versi ini:

- **Crash saat streaming dunia** — ditambah mutex thread-safe di mapping resource Direct3D 9, jadi tidak mati saat menyetir dan tekstur dimuat di belakang.
- **Watchdog 5 menit dihapus** — sebelumnya game nutup sendiri sekitar 4,5 menit karena timeout saat bring-up CRT.
- **Draw distance 200 m plus kabut jarak** — tuning supaya frame rate stabil.
- **Bahasa default Spanyol**, bahasa lain (termasuk Inggris) bisa diganti di `nfsu2.cfg`.

Klaim performanya sekitar **25–30 FPS** di RK3326. Buat chip sekelas itu, angka yang wajar untuk game balap 2004.

## Cara pasang

Tiga langkah, semuanya di SD card:

1. Extract `NFSU2-R36S.zip` ke folder `ports/` di SD card.
2. Copy file game PC yang kamu punya sendiri ke `ports/nfsu2/gamedata/` — isinya `SPEED2.EXE`, lalu folder `GLOBAL/`, `CARS/`, `TRACKS/`, `SOUND/`, `MOVIES/`.
3. Colok SD ke konsol, buka menu **Ports**, jalankan **NFSU2**.

## Yang perlu dicatat sebelum download

- **Zip-nya bukan game lengkap.** Isinya engine hasil recompile saja. Asset — mobil, trek, musik — tetap harus dari salinan PC legal milik sendiri. Tanpa itu game tidak jalan.
- **Biner dari komunitas, bukan EA.** Tidak ada tanda tangan digital, tidak lewat toko aplikasi. Wajar untuk port rumahan, tapi tetap: jalankan di SD card terpisah, bukan di card yang isinya data penting.
- **Klaim FPS belum aku uji sendiri.** Aku belum punya unitnya, jadi angka 25–30 FPS itu dari catatan rilis, bukan hasil pengukuran di tangan.

Kalau ada yang sudah coba di R36S, kabari di komentar — penasaran apakah frame drop-nya kelihatan saat balapan malam di area kota yang ramai.

Sumber: [rilis v1.0.0-r36s](https://github.com/manuelpllull/nfsu2-recompiled-r36s/releases/tag/v1.0.0-r36s-20261017-c325ba2)
