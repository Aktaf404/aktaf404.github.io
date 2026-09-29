---
title: qBittorrent vs Pi-hole, Perang Port 8080
date: 2026-09-29 19:00:00 +0700
categories: [Homelab, CasaOS]
tags: [docker, qbittorrent, pihole, casaos]
---

qBittorrent di CasaOS gw gak bisa dibuka. Buka `http://192.168.18.16:8080` → **403**. Pikir error qBittorrent — bukan. Itu **Pi-hole**.

## Kenapa

qBittorrent pake `network_mode: host`. Konsekuensinya: **port mapping di compose diabaikan**, qB bind langsung `*:8080`.

Pi-hole map `80/tcp → 8080` di host. Docker-proxy Pi-hole duluan pegang port → qBittorrent:

```
WebUI: Unable to bind to IP: *, port: 8080. Reason: The bound address is already in use
```

## Fix

Image hotio override `WebUI\Port` dari env `WEBUI_PORTS`. Edit config sendiri gak cukup — entrypoint overwrite pas start.

```yaml
environment:
    WEBUI_PORTS: 8181/tcp
```

Lalu:

```bash
cd /var/lib/casaos/apps/qbittorrent
docker compose down && docker compose up -d
```

Tunggu ~8 detik, log bakal nunjukin:

```
To control qBittorrent, access the WebUI at: http://localhost:8181
```

Password temporary muncul di log **sekali**. Langsung ganti lewat WebUI → Options → WebUI.

## Pelajaran

`network_mode: host` = port di compose **dekorasi**. Selalu `ss -tlnp | grep <port>` sebelum set WebUI ke port yg sama deng applikasi lain.
