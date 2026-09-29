---
title: Tailscale Funnel, VPS Gratis Selamanya
date: 2026-09-28 20:00:00 +0700
categories: [Homelab, Networking]
tags: [tailscale, funnel, selfhosting]
---

Semua service di mini PC gw bisa diakses dari luar rumah tanpa port forwarding, tanpa IP publik, tanpa VPS.

## Caranya

Tailscale Funnel. 1 perintah:

```bash
tailscale funnel --https=8443 3000
```

Selesai. `https://srv-01.tailb504f8.ts.net:8443` langsung jalan, HTTPS valid, sertifikat otomatis.

## Yang gw expose

- `:8443` — 3D HUD mini PC
- `:10000` — Portfolio
- `:8443/pjlp` — PJLP dump
- `:443` — Hermes dashboard

## Catatan teknis

- Funnel cuma terima target **localhost**. Service di mesin lain (misal LXC) harus di-forward duluan ke `127.0.0.1` pake `socat`, baru funnel ke port lokal itu
- Funnel state **server-side**. Ganti hostname = entry lama nyangkut. Fix: `tailscale funnel reset` lalu re-apply satu per satu pake jeda 5-6 detik (`etag mismatch` kalau serempak)
- Sertifikat re-issue butuh **~30 detik**. Jangan salah diagnosa sbg funnel gagal
- Dari HP: kadang timeout soalnya operator LTE Indonesia sering putusin relay DERP. **Matikan Tailscale app di HP** — funnel tetep resolve via DNS publik

## Biaya

Rp 0. Tailscale personal plan, 100 device, funnel unlimited.
