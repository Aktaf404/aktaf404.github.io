---
title: Mati Lampu Semalam, 4 Funnel Auto-Pulih
date: 2026-09-29 01:00:00 +0700
categories: [Homelab, Recovery]
tags: [proxmox, tailscale, systemd, funnel]
---

Malam 28 September 2026, listrik rumah mati. Mini PC `srv-01` ikut mati. Besoknya hidup lagi — dan 3 dari 4 Tailscale Funnel langsung jalan sendiri tanpa dipegang.

Kenapa bisa? Karena tiap funnel service punya `ExecStartPre` **readiness loop**:

```bash
ExecStartPre=/bin/bash -c 'for i in $(seq 1 30); do tailscale status >/dev/null 2>&1 && break; sleep 2; done'
```

## Masalah aslinya

Sebelumnya `ExecStartPre=/bin/sleep 5`. Itu cuma nunggu **daemon**, bukan **tailnet**. Hasilnya pas boot, funnel langsung jalan → `tailscaled` masih `NoState` → semua funnel **failed**.

`After=tailscaled.service` juga gak cukup — systemd anggap "siap" begitu proses jalan, padahal tailnet belum connect.

## 1 service yg kena

`hermes-dashboard-funnel` belum di-patch. Begitu nyala, langsung failed. Patch pakai loop yg sama → idup.

## Pelajaran

- Funnel = butuh tailnet ready, bukan daemon ready
- Systemd `After=` gak menunggu service "siap" secara fungsi
- Test: stop semua funnel, start berurutan, lihat semua idup. Simulasi lebih valid daripada restart
- Backup config sebelum patch: `*.service.bak`

Besok kalo mati lampu lagi, tinggal cek: semua funnel harus `active`. Kalau ada yg failed, loop-nya gak ditaruh.
