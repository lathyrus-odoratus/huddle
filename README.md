# Huddle

社群聚會的簽到、破冰與排程小工具。以 **Discord Activity** 為主要入口，也可以直接用瀏覽器開啟。

- **系列活動**：用 Series 串起每一場聚會（#1、#2…），用邀請連結加入
- **出席與簽到**：事前表態「參加 / 可能 / 不參加」，活動進行中按下簽到
- **破冰盲揭**：每場一題，回答在揭曉前只有自己看得到；主持人逐一揭曉，也可以「隨機下一個」，所有人一起看轉盤動畫
- **下次時間**：成員勾選候選時間或自由填寫，主辦鎖定後自動建立下一場
- **即時互動**：誰在線上、誰剛簽到、表情反應即時同步
- **歷史回顧**：在 Series 裡左右翻閱過去的場次

## 目前狀態

> **階段：規格定案，尚未開始實作**（2026-10-03）

| 里程碑 | 狀態 |
| --- | --- |
| 需求與規格討論 | ✅ 完成 |
| Repo 與文件 | ✅ 完成 |
| 機密與環境變數管理方式 | ⏳ 待決定 |
| 專案骨架（pnpm workspace、Worker、Vue、shared） | ⬜ |
| Cloudflare 環境（staging / production、D1、網域） | ⬜ |
| Discord Applications（Staging / Production） | ⬜ |
| 登入（Activity + 網頁） | ⬜ |
| Series / 邀請 / 成員 | ⬜ |
| Event 生命週期、出席與簽到 | ⬜ |
| 即時房間（Durable Object） | ⬜ |
| 破冰盲揭 | ⬜ |
| 意願收集與鎖定 | ⬜ |
| 個人狀態頁、Series 場次翻閱 | ⬜ |
| Admin 管理頁、`/huddle openwebpage` | ⬜ |
| CI/CD | ⬜ |

## 技術棧

| 層 | 技術 |
| --- | --- |
| 前端 | Vue 3、Vite、TypeScript、Pinia、TanStack Query、Tailwind CSS v4、shadcn-vue、motion-v、vue-i18n |
| 後端 | Cloudflare Workers、Hono |
| 資料 | Cloudflare D1、Drizzle ORM |
| 即時 | Cloudflare Durable Objects（WebSocket Hibernation） |
| Discord | Embedded App SDK、OAuth2、HTTP Interactions |

## 環境

| 環境 | 網址 |
| --- | --- |
| production | https://huddle.miao-bao.cc |
| staging | https://huddle-staging.miao-bao.cc |

尚未部署。

## 文件

- [技術規格](docs/spec.md)
- [初始需求](docs/initial-requirements.md)
- 架構決策（ADR）
  - [0001 採用 Cloudflare 全家桶與 Vue 3](docs/adr/0001-cloudflare-and-vue.md)
  - [0002 獨立 Web App，Activity 為入口之一；不做主動推播](docs/adr/0002-standalone-webapp-with-activity-entry.md)
  - [0003 REST 負責寫入、DO 負責房間協調](docs/adr/0003-rest-writes-do-coordination.md)
  - [0004 Global 身份、系統自管權限、邀請制加入](docs/adr/0004-global-identity-and-invite-only.md)
  - [0005 主辦與主持人分離](docs/adr/0005-organizer-facilitator-separation.md)
  - [0006 Event 手動生命週期與破冰盲揭](docs/adr/0006-manual-event-lifecycle-and-blind-reveal.md)
  - [0007 staging 與 production 環境](docs/adr/0007-staging-and-production-environments.md)
