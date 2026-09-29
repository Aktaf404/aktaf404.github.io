---
title: TanStack CLI, MCP Server yg Sudah Dihapus
date: 2026-09-30 09:30:00 +0700
categories: [Web, AI]
tags: [tanstack, mcp, cli, ai]
---

Mau pasang MCP server TanStack biar agent gw bisa query docs + scaffold aplikasi. Setelah diulik, **MCP server-nya udah dihapus** resmi.

## Yang terjadi

 di `TanStack/cli` (repo resmi) ada folder `mcp`. Tapi pas dibaca docs-nya:

> *"`tanstack mcp` has been removed from the CLI and will not be restored."*

MCP server emang pernah ada, tapi sekarang **docs migrasi** — semua fungsi dipindah ke CLI biasa.

## Kenapa ini menarik

Semua pembahasan MCP selalu bikin sound seperti "agent harus connect ke MCP server". Realitanya: vendor bisa hapus kapan aja. Tools CLI biasa dgn output **JSON** sama kuatnya, dan gak akan ilang.

## Cara pake (pengganti MCP)

```bash
tanstack libraries --json
tanstack create --list-add-ons --framework React --json
tanstack create --addon-details drizzle --framework React --json
tanstack search-docs "server functions" --library start --framework react --json
tanstack doc query framework/react/overview --json
tanstack ecosystem --category auth --json
```

Semua support `--json` — output deterministic, gampang diparse agent. Ini level feature parity dgn MCP tools yg lama:

| Old MCP Tool | New CLI Command |
|---|---|
| `listTanStackAddOns` | `tanstack create --list-add-ons --json` |
| `getAddOnDetails` | `tanstack create --addon-details drizzle --json` |
| `createTanStackApplication` | `tanstack create my-app --add-ons ...` |
| `tanstack_list_libraries` | `tanstack libraries --json` |
| `tanstack_doc` | `tanstack doc query ... --json` |
| `tanstack_search_docs` | `tanstack search-docs "..." --json` |
| `tanstack_ecosystem` | `tanstack ecosystem --category auth --json` |

## Scaffold aplikasi

```bash
npx @tanstack/cli create my-app                    # full-stack TanStack Start
npx @tanstack/cli create my-app --blank -y         # minimal, no prompts
npx @tanstack/cli create my-app --router-only      # SPA, no SSR
npx @tanstack/cli create my-app --add-ons clerk,drizzle,tanstack-query
npx @tanstack/cli add clerk drizzle                # add to existing project
```

## Yang masih ada: MCP add-on

Ada `mcp` **add-on** — tapi itu **berbeda**. Itu bikin aplikasi TanStack Start lu sendiri nge-host MCP endpoint. Server feature dari app yg lu build, bukan tool buat agent lu.

Jangan tertukar:
- ~~`tanstack mcp`~~ — MCP server buat agent → **dihapus**
- `--add-ons mcp` — MCP server dalem app lu → **masih ada**

## Pelajaran

- Repository nyari keyword "mcp" ketemu → belum tentu feature itu masih idup. Bisa jadi itu **migration docs** yg nunjukin cara pindah
- CLI + `--json` = alternatif MCP yg stabil. Vendor gak gampang hapus CLI
- Cek repo tree resmi (`git/trees/main?recursive=1`) sebelum asumsi feature ada
