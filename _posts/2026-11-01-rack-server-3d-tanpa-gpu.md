---
title: "Rack Server Homelab 3D di Server Tanpa GPU: Cuma Kode Three.js, Nol Install Berat"
date: 2026-11-01 12:35:00 +0700
categories: [Homelab, 3D]
tags: [threejs, webgl, swiftshader, render, tanpa-gpu]
image: /assets/img/rack-server-3d.png
---

Pengin lihat rack homelab-ku dalam bentuk 3D — bukan foto, tapi model yang bisa diputar, kipasnya muter, LED-nya nyala. Masalahnya server-ku cuma Intel i3-8100T tiga core, 15 GB RAM, dan **nggak ada GPU sama sekali** yang bisa dipakai.

Bukan karena nggak ada kartu grafis. `lspci` jelas nyebut ada Intel UHD Graphics 630. Tapi ini container Proxmox:

```bash
$ ls /dev/dri
ls: cannot access '/dev/dri': No such file or directory
```

Device GPU-nya nggak diteruskan ke container. Jadi mau render 3D lewat GPU: nggak mungkin. Yang aku tahu, bikin 3D tanpa GPU itu urusan Blender Cycles yang bakal jalan semalam suntuk, atau nggak jalan sama sekali.

Ternyata bisa, dan cepat. Ini hasilnya.

## Hasilnya dulu

<video controls playsinline preload="metadata" style="max-width:100%;border-radius:8px">
  <source src="/assets/files/rack-server-3d-turntable.mp4" type="video/mp4">
</video>

Coba sendiri langsung di browser: **[demo rack 3D](/assets/rack3d/index.html)** — drag buat muter, scroll buat zoom. Jalan di HP juga.

Spesifikasi model, semuanya angka nyata dari scene:

| Item | Nilai |
|---|---|
| Dimensi rack | 0,60 × 2,02 × 1,07 meter (42U, open frame) |
| Jumlah mesh | 906 |
| Triangle | ~29 ribu |
| Server | 13 chassis (1U, 2U, 4U campur) |
| Kipas | 7 unit, semua berputar |
| Berat file | 265 KB (zip, sudah termasuk Three.js) |
| GPU dipakai | nol |

## Kenapa bisa jalan tanpa GPU

Dua hal yang sering ketuker.

**Pertama: modelnya matematika, bukan render.** Rack ini dibangun dari perintah seperti `new THREE.BoxGeometry(0.44, h, 0.72)` lalu ditaruh di posisi tertentu. Servernya ya kotak, kipasnya ya shape yang diputar 7 kali. Nggak ada file mesh yang diunduh, nggak ada fotogrametri, nggak ada neural network yang nebak bentuk. Hasilnya: satu file `index.html` plus Three.js 1,27 MB.

Karena semuanya angka, membuat modelnya nol biaya GPU. GPU baru kepakai saat **menggambar** ke layar.

**Kedua: yang menggambar bisa CPU.** Chromium bisa menjalankan WebGL tanpa kartu grafis lewat **SwiftShader** — rasterizer software buatan Google, jalan di CPU. Aku jalankan headless Chromium dengan flag itu:

```bash
chrome --use-gl=angle --use-angle=swiftshader --enable-unsafe-swiftshader
```

Kalau dicek dari dalam halaman, renderer yang aktif bukan GPU:

```text
ANGLE (Google, Vulkan 1.3.0 (SwiftShader Device (Subzero) (0x0000C0DE)), SwiftShader driver)
```

Itu tanda semua pixel dihitung di CPU.

**Ongkosnya:** lambat. Video turntable 90 frame ukuran 760×940 butuh sekitar 2–3 menit. Screenshot 1120×1120 kira-kira 6 detik per frame. Dengan GPU asli, pekerjaan yang sama selesai dalam hitungan detik. Tapi buat bikin gambar dan video pendek, sabar sedikit jauh lebih murah daripada beli GPU.

## Isi rack-nya

Biar nggak cuma kotak hitam, aku detailkan bagian yang biasanya bikin model 3D kelihatan palsu:

- **Frame:** 6 post vertikal, panel samping berventilasi, panel belakang dengan slat grille, plafon, dan plinth bawah.
- **Rails:** dua rel depan dengan pola lubang U — 3 lubang per 1U, jarak 14,8 mm antar baris, seperti rack EIA asli.
- **Server:** tiap chassis punya bezel, drive bay dengan handle, slot ventilasi, LED status hijau/biru, strip label, tombol power, dan kuping mounting kiri-kanan.
- **Kipas:** 3 exhaust di atas, 2 intake di bawah, 2 exhaust di belakang. Tiap kipas punya housing, ring pelindung, hub, dan 7 bilah melengkung dari `ExtrudeGeometry` dengan pitch 0,32 radian. Rotor belakang berputar arah berlawanan.
- **Kabel:** 12 jalur di belakang pakai `CatmullRomCurve3` lalu dijadikan `TubeGeometry`, biar ada kesan "ini rack yang benar-benar dipakai".
- **Cahaya:** hemisphere light, key light dengan bayangan, rim light biru, plus environment map dari `RoomEnvironment` — tanpa itu, material metalik (metalness 0,9) dirender hitam total. Ini kesalahan pertamaku: modelnya kelihatan seperti siluet.

## Verifikasi, bukan cuma "kelihatan bagus"

Model 3D gampang bikin mata tertipu. Jadi aku tulis satu skrip pemeriksa yang jalan di browser headless dan menolak hasil yang melanggar batas:

```text
fails: []
spin:  {'rotorsMoved': 7, 'total': 7}
extent: {'minY': -0.0, 'maxY': 2.02, 'maxX': 0.301, 'depth': 1.07}
```

Yang dicek: nggak ada geometri yang tembus lantai, nggak ada yang keluar dari lebar rack, seluruh 7 rotor benar-benar berputar (bukan cuma ada di kode), framing kamera nggak terpotong, dan nol error di console.

Zip-nya juga aku uji ulang dengan cara diekstrak ke folder bersih lalu dijalankan dari port lain — hasilnya tetap `fails: []`. Jadi kalau file itu yang kamu download, isinya memang yang sudah lulus uji.

## Batas jujur

Model ini **bukan replika** rack merek tertentu. Aku bikin tanpa foto referensi, jadi proporsinya ikut standar 19 inci dan tampilannya gaya homelab pada umumnya. Bagian yang nggak kelihatan dari satu sudut — kabel belakang, isi dalam chassis — aku karang sendiri, bukan hasil pengukuran.

Kalau tujuannya meniru objek dari foto dengan ketat, jalurnya beda: ada tool `img2threejs` yang memaksa alur ketat — validasi foto, assessment kompleksitas, tulis spec, bangun per pass, lalu bandingkan render dengan foto pakai skor per layer. Itu butuh foto referensi dan jauh lebih banyak langkah. Untuk rack generik, cara prosedural seperti ini sudah cukup.

Catatan teknis lain: render di CPU itu lambat. Kalau nanti mau animasi 60 fps atau resolusi 4K, di situlah GPU benar-benar dibutuhkan.

## Ambil filenya

- **[Coba demo 3D-nya](/assets/rack3d/index.html)** — buka di browser, langsung jalan
- **[Download zip](/assets/files/rack-server-3d.zip)** (265 KB) — `index.html` + Three.js, offline-friendly
- **[Download video turntable](/assets/files/rack-server-3d-turntable.mp4)** (848 KB, 6 detik)

Cara jalanin dari zip:

```bash
unzip rack-server-3d.zip -d rack3d
cd rack3d
python3 -m http.server 8099
# buka http://localhost:8099
```

Perlu server lokal, bukan dobel-klik file-nya, karena ES module di browser nggak bisa dimuat lewat `file://`.

Selanjutnya yang mau aku tambah: klik server buat lihat nama dan perannya (pve-01, NAS, switch), lalu versi 12U wall-mount yang lebih pas buat rak dinding kamar. Yang jelas, nggak ada satu pun langkah di atas yang butuh GPU.
