---
title: "WebBrain: Bikin Chrome Headless Jadi Browser Agent di Server Tanpa Monitor"
date: 2026-10-13 22:00:00 +0700
categories: [Homelab, AI]
tags: [webbrain, chrome, headless, mcp, hermes]
---

Server homelab aku itu Proxmox container tanpa monitor. Nah pas nemu [WebBrain](https://github.com/webbrain-one/webbrain) — browser agent open-source (GPL-3.0) yang bisa dikontrol lewat MCP — kepikiran: bisa gak ya jalanin ini di server, browser-nya headless?

Jawabannya: bisa, tapi ada dua jebakan. Tulisan ini catatan cara aku lewatinnya.

## Apa itu WebBrain

WebBrain itu ekstensi browser AI agent: dia bisa baca halaman, klik, isi form, dan jalanin task multi-langkah. Biasanya dipakai di laptop — pasang ekstensinya di Chrome, buka side panel, kasih task.

Yang bikin menarik buat homelab: dia punya **MCP server**. Skemanya begini:

```
Hermes ──stdio──▶ webbrain-mcp ──ws://127.0.0.1:17374──▶ ekstensi WebBrain ──▶ tab browser
```

Jadi agent lain (dalam kasusku Hermes) bisa delegasikan task browser ke ekstensi WebBrain. Poin pentingnya: arah koneksinya kebalik dari yang kubayangkan. Ekstensi Chrome gak bisa *listen* port (keterbatasan Manifest V3), jadi MCP server yang *host* listener, dan ekstensi yang *dial out* ke dia.

## Jebakan 1: Chrome branded menolak `--load-extension`

Rencana awal simpel: Chrome headless + `--load-extension`.

```bash
google-chrome --headless=new --load-extension=/opt/webbrain/src/chrome ...
```

Chrome-nya jalan, log bersih, tapi ekstensi **tidak pernah muncul**. Cek `Default/Preferences` di profil: yang ke-load cuma component extensions bawaan. Path ekstensi? Tidak ada sama sekali.

Ternyata Google memblokir `--load-extension` di **Chrome 137+** versi branded. Bukan error, bukan warning — flag-nya diabaikan diam-diam. Sempat tergiur pakai CDP buat cek isi ekstensi, tapi DevTools server Chrome-nya malah kena `CreatePlatformSocket() failed: Permission denied` di container ini (Python bisa bikin socket normal, jadi quirk sandbox Chrome sendiri — dan untungnya tidak dibutuhkan).

Solusinya: pakai **Chromium build** dari `@puppeteer/browsers`:

```bash
npx -y @puppeteer/browsers install chromium@latest --path /opt/chromium-builds
/opt/chromium-builds/chromium/linux-1717541/chrome-linux/chrome \
  --headless=new \
  --user-data-dir=/tmp/wb-chrome-profile \
  --load-extension=/opt/webbrain/wb-headless-ext \
  --no-first-run --disable-gpu --no-sandbox \
  --disable-dev-shm-usage \
  about:blank &
```

Setelah ganti binary, ekstensi langsung ke-load. Terlihat dari ID ekstensi yang terdaftar di Preferences.

## Jebakan 2: Bridge perlu "diklik" dulu

Cara normal pakai WebBrain: pasang ekstensi, buka **Settings → Bridge**, isi `ws://127.0.0.1:17374/extension`, klik enable. Di server tanpa monitor, tidak ada yang bisa klik.

Untung setelah baca source-nya, setting itu cuma dua key di `chrome.storage.local`, dan ekstensi membaca key yang sama setiap startup. Patch-nya satu baris di `src/cloud-runs.js`, fungsi `syncBridge()`:

```js
async function syncBridge() {
  const stored = await api.storage.local.get(['webbrainCloudBridgeEnabled', 'webbrainCloudBridgeUrl']);
  // sebelumnya: kalau enabled-nya false, bridge di-stop. Patch: selalu start.
  return startBridge(stored.webbrainCloudBridgeUrl || DEFAULT_CLOUD_BRIDGE_URL);
}
```

Aku bikin salinan ekstensi, patch, dan arahkan `--load-extension` ke salinan itu. Hasilnya begitu Chrome hidup, bridge langsung nyambung sendiri.

## MCP server-nya sendiri

Build dari source (repo-nya monorepo, MCP server ada di `mcp-server/`):

```bash
cd /opt/webbrain/mcp-server
npm install && npm run build
node dist/index.js
```

Enam tools yang tersedia:

| Tool | Fungsi |
|---|---|
| `webbrain_run` | Delegasikan task browser. `mode='ask'` read-only, `mode='act'` boleh klik/ketik |
| `webbrain_extract` | Baca halaman jadi JSON sesuai schema |
| `webbrain_status` | Cek status run |
| `webbrain_respond` | Jawab run yang lagi nunggu keputusan |
| `webbrain_abort` | Batalkan run |
| `webbrain_connection` | Cek ekstensi nyambung atau enggak |

## Tes end-to-end

Satu hal yang kusukai: permission gate-nya tetap aktif walau headless. Saat aku minta buka `example.com`, run-nya berhenti dan minta persetujuan dulu — `network_write` ke host itu, jawabannya cuma boleh `once` / `always` / `deny`. Setelah disetujui, run lanjut dan jawab bener:

> **Page title:** Example Domain
> **First heading:** Example Domain

## Catatan operasional

Beberapa hal yang perlu diingat kalau mau replikasi:

- **Port 17374 cuma boleh dipegang satu proses.** MCP server bind `127.0.0.1:17374` saat start; kalau ada sisa proses lama (atau sesi MCP client lain yang masih hidup), proses baru langsung mati dengan `EADDRINUSE` yang kelihatan sebagai "Connection closed". `kill $(lsof -ti:17374)` dulu sebelum test.
- **Ekstensi reconnect otomatis** dengan backoff eksponensial (maksimal 30 detik). Kalau MCP server di-restart, tunggu saja sebentar, jangan panik.
- **`webbrain_respond` pakai argumen snake_case** (`run_id`, `clarify_id`, `answer`) — beda dari kebiasaan camelCase tools lain.
- MCP server stdio keluar kalau stdin-nya ditutup. Buat mode daemon, kubuatkan wrapper kecil yang menahan stdin lewat fifo — sekalian jadi jalur kirim perintah.

## Kesimpulan

Total setup selesai dalam satu sesi, tanpa monitor dan tanpa mengorbankan permission gate. Ekstensi WebBrain headless di Chromium build + MCP server lokal = agent browser yang bisa dipanggil dari mana saja di jaringan internal.

Repo: [webbrain-one/webbrain](https://github.com/webbrain-one/webbrain) — GPL-3.0, ada di Chrome Web Store / Firefox Add-ons / Edge Add-ons juga kalau mau versi desktop.
