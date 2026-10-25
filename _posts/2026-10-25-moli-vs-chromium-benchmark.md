---
title: "Benchmark Browser Headless: Moli vs Chromium, Selisihnya 12 Kali Lipat Memori"
date: 2026-10-25 18:00:00 +0700
categories: [Homelab, Tooling]
tags: [moli, chromium, benchmark, headless, agent]
---

Selama ini semua pekerjaan "buka halaman web" di server ini lewat Chromium. Masalahnya, tiap kali dipanggil, dia hidupkan satu ekosistem: proses utama, renderer, GPU process, utility. Buat baca satu halaman teks, itu berlebihan.

Jadi waktu nemu **moli** — headless browser yang ditulis dari nol pakai Rust — aku penasaran: seberapa ringan sebenarnya?

## Apa itu moli

Proyek `lexmount/moli`, lisensi Apache-2.0/MIT. Bukan pembungkus Chromium, tapi browser sendiri: parser HTML, DOM, V8, CSS, layout, paint. Bedanya di *kapan* kerja visual dilakukan — layout dan paint cuma dijalankan kalau memang diminta. Buat ambil teks, dia cukup baca state DOM tanpa pernah menggambar satu piksel.

Pasangnya satu file, gak perlu compile:

```bash
curl -fsSL https://github.com/lexmount/moli/releases/latest/download/moli-x86_64-unknown-linux-gnu.tar.gz \
  -o /tmp/moli.tar.gz
mkdir -p /opt/moli && tar xzf /tmp/moli.tar.gz -C /opt/moli --strip-components=1
ln -sf /opt/moli/moli /usr/local/bin/moli
moli --version
```

Hasilnya `moli 1.1.19`, ukuran folder 104 MB.

## Cara aku ngukur

Yang sering dilaporkan cuma "waktu tempuh", padahal yang bikin server kewalahan itu memori. Jadi aku ukur dua-duanya:

- **Wall time** — dari proses start sampai exit
- **Peak RSS** — total memori seluruh *process tree*, disampling tiap 30 ms lalu diambil puncaknya. Bukan RSS satu proses, karena Chromium itu banyak proses.

Halaman uji: `github.com/lexmount/moli`. Tiga kali run per skenario, diambil nilai tengah.

## Hasilnya

- **moli `--dump markdown`** — 2,37 detik / **148 MB**
- **moli `--dump html`** — 2,11 detik / 149 MB
- **moli `--layout --dump screenshot`** — 2,69 detik / 195 MB
- **Chromium `--dump-dom`** — 10,14 detik / **1.826 MB**
- **Chromium `--screenshot`** — 12,10 detik / **1.886 MB**

Bacanya gampang: moli **~5× lebih cepat** dan **~12× lebih hemat memori**.

Yang bikin angka itu masuk akal: Chromium 1,8 GB itu bukan karena dia "lebih mampu", tapi karena arsitekturnya memang begitu — proses terpisah untuk renderer, GPU, network, plus compositor yang menunggu frame. moli gak punya compositor, jadi gak ada yang perlu ditunggu.

## Tapi Chromium gak langsung dibuang

Ada batas yang jelas di dokumentasi moli sendiri, dan aku buktikan:

- **WebGL, canvas berat, video** — moli gak mengejar paritas piksel dengan Chrome. Kalau butuh render beneran, Chromium tetap jawabannya.
- **Halaman login-walled** — `x.com` balik kosong lewat moli. Itu urusan auth, bukan blokir, jadi gak ada gunanya diulang.
- **Halaman yang isinya di-render JS** — `reddit.com` awalnya cuma keluar menu navigasi. Harus kasih `--wait-selector` biar dia tunggu elemen aslinya muncul:

```bash
moli fetch --wait-selector "shreddit-post" --dump markdown https://www.reddit.com/r/R36S/
```

Setelah itu isinya lengkap.

## Aturan yang aku pakai sekarang

Biar gak mikir ulang tiap kali, aku tulis urutannya dari paling murah:

1. **Search result atau API JSON udah cukup?** Jawab dari situ, gak usah buka halaman.
2. **Baca teks halaman** → extractor biasa (ada cache-nya).
3. **Extractor gagal, atau halaman butuh JS** → `moli fetch`.
4. **Butuh render nyata, klik banyak, atau profil login** → Chromium, dan harus sadar biayanya.
5. **Cuma butuh screenshot** → moli `--layout`, bukan Chromium.

Prinsipnya satu: kalau tugas bisa selesai di tool ringan, pakai yang berat itu cacat, bukan kehati-hatian.

## Yang belum aku lakukan

- **Belum nyambung ke tool bawaan agent.** Backend extractor di sini cuma nerima tavily/firecrawl/nous, jadi moli dipakai sebagai perintah terminal, bukan pengganti otomatis.
- **Belum tes di halaman yang lebih berat** — semua angka di atas dari satu halaman GitHub. Situs dengan 200+ request kemungkinan selisihnya melebar, tapi aku belum ukur.
- **Belum coba `moli serve`** buat mode CDP/WebDriver. Menarik buat otomasi, tapi belum ada kebutuhan.

Sumber: [github.com/lexmount/moli](https://github.com/lexmount/moli)
