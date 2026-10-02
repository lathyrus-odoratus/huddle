# ADR 0001：採用 Cloudflare 全家桶與 Vue 3

- 狀態：已接受
- 日期：2026-10-03

## 背景

Huddle 以 Discord Activity 為主要入口。Activity 在 Discord proxy 後的 iframe 中執行，CSP 嚴格，前端最好只連自己的單一網域。OAuth code 交換需要後端。另外需要即時多人互動（線上名單、簽到廣播、破冰揭曉、表情反應）。

## 決策

- 後端：一個 **Cloudflare Worker**（Hono），同時提供 Vue SPA（Static Assets）、REST API、WebSocket、Discord interactions。
- 資料：**D1**（SQLite）+ Drizzle ORM。
- 即時：**Durable Objects**，每個 Event 一個房間，使用 WebSocket Hibernation。MVP 就導入。
- 前端：**Vue 3 + Vite + TypeScript**，Pinia、TanStack Query、Tailwind v4、shadcn-vue、motion-v、vue-i18n。

## 考慮過的替代方案

- Node（Express/Fastify）+ Postgres + discord.js，部署於 Fly.io/Railway：需要自行維護常駐 process 與 DB。
- Supabase：前端直連會被 Activity 的 CSP 擋下，必須全部經 proxy，失去 BaaS 的主要優勢。

## 影響

- 單一 repo、單一部署單位，前後端版本一致；成本近乎零，沒有主機要維護。
- Activity 的 URL Mapping 只需一條。
- 受限於 Workers runtime（無 Node 原生模組；D1 為 SQLite 語意）。
