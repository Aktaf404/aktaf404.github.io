---
title: 9router, 5 Key Atria Round-Robin
date: 2026-09-29 20:00:00 +0700
categories: [Homelab, AI]
tags: [9router, atria, proxy, api]
---

Atria itu API key AI gw. Tapi 1 key = quota kecil. Solusinya: **9router** di HP, rotate 5 key sekaligus.

## Arsitekturnya

```
Hermes/Opencode → 9router (HP, port 20128) → Atria (5 key round-robin)
```

Bukan gw pilih 9router soalnya dia murah. Dia **ngefilter request** yg Atria tolak. Contoh: Hermes kirim `tools: []` (array kosong). Atria direct tolak:

```
400: `tools` must not be an empty array
```

9router filter itu sebelum diteruskan. **Direct ke Atria = gagal, lewat 9router = jalan.**

## Pool 5 key

| Nama | Prioritas |
|---|---|
| gmail-7 | 0 (utama) |
| gmail-3 | 1 |
| gmail-4 | 1 |
| gmail ke 5 | 1 |
| gmail ke 6 | 1 |

Semua model `Atria-Dawn-Preview`, provider `openai-compatible-responses`.

## Setup via database

Key disimpen di SQLite, bukan file config:

```bash
sqlite3 /root/.9router/db/data.sqlite \
  "SELECT name, priority FROM providerConnections"
```

Tambah key = insert row dgn `provider=openai-compatible-responses-<uuid>`, `data` JSON berisi `apiKey`, `defaultModel`, dan `providerSpecificData.baseUrl`. Lalu restart 9router.

**Hati-hati duplicate key** — gak ada error, tapi quota kebagi 2 pool. Cek:

```bash
SELECT name, apiKey FROM providerConnections
```

## Kenapa pindah key utama

Key baru `gmail-7` diprioritaskan (`priority=0`), lainnya `1`. Pas satu key kena rate limit, 9router otomatis pake key lain tanpa putus.

## Pelajaran

- Proxy kecil = nyawa. Filter 1 validasi bisa bedain jalan/gak
- Prioritas 0 = dipakai duluan; selain itu fallback
- Cek `lastUsedAt` buat pastiin key beneran kepake, bukan cuma ke-insert
