# 12-Factor 實踐狀況

以 [The Twelve-Factor App](https://12factor.net/) 檢視 Huddle 的設計與實作。Huddle 跑在 Cloudflare Workers 這種 serverless 平台上，部分 factor 由平台處理，表中會說明它在這裡對應的意義。

> 最後更新：2026-10-03（規格階段，尚未實作）
> 相關決策：[ADR 0008](./adr/0008-twelve-factor-adjustments.md)、[ADR 0009](./adr/0009-secrets-via-github-environments.md)

## 圖例

| 標記 | 意義 |
| --- | --- |
| 🟢 | 完全符合 |
| 🟡 | 符合，但有已知的妥協 |
| 🔵 | 由平台處理 |
| ⬜ | 已設計，尚未實作 / 驗證 |

## 總覽

| # | Factor | 設計 | 實作 |
| --- | --- | :-: | :-: |
| I | Codebase | 🟢 | ⬜ |
| II | Dependencies | 🟢 | ⬜ |
| III | Config | 🟡 | ⬜ |
| IV | Backing services | 🟢 | ⬜ |
| V | Build, release, run | 🟢 | ⬜ |
| VI | Processes | 🟢 | ⬜ |
| VII | Port binding | 🔵 | — |
| VIII | Concurrency | 🔵 | — |
| IX | Disposability | 🟢 | ⬜ |
| X | Dev/prod parity | 🟡 | ⬜ |
| XI | Logs | 🟢 | ⬜ |
| XII | Admin processes | 🟢 | ⬜ |

「實作」欄會在對應功能完成時更新，並附上驗證方式。

---

## I. Codebase 🟢

一個 repo（`lathyrus-odoratus/huddle`），部署為 staging 與 production 兩套。前端、Worker、共用協定在同一個 pnpm workspace。

## II. Dependencies 🟢

- 所有依賴（包含 `wrangler`）都宣告在 `package.json` 並由 `pnpm-lock.yaml` 鎖版本。
- CI 不使用全域安裝或未鎖版本的 `npx`。

## III. Config 🟡

**符合的部分**
- 前端 build 不含任何環境設定，啟動時從 `GET /api/config` 取得公開設定。所有環境部署同一份前端產物。
- 應用設定與機密從環境注入 Worker（來源為 GitHub Environments，見 ADR 0009）。
- 機密不進版控；本機使用 `.dev.vars`。

**妥協**
- `wrangler.jsonc` 中各環境的區塊（網域、D1 database ID、binding）是放在 repo 裡的。這是 Workers 的慣例：這些屬於**基礎設施拓撲**而不是應用設定，也不是機密，所以接受。
- 本機與 CI 的設定是兩份資料，需要人工保持一致。

## IV. Backing services 🟢

D1、Durable Objects、Discord API 都透過 binding 或 URL 接入，換環境只需改設定，不需改程式碼。

## V. Build, release, run 🟢

- **Build**：每個 commit 只 build 一次（Worker bundle + web `dist`），存成 CI artifact。
- **Release**：artifact + 環境設定 → 部署。每次部署產生不可變的 Worker version ID，可用 `wrangler rollback` 回退。
- **Run**：Cloudflare 執行。
- production 只能部署已在 staging 部署過的 commit。
- Migration 採 expand / contract，確保 rollback 後舊程式碼仍能面對新 schema。

⬜ 待驗證：預先 build 的 bundle 部署到不同環境的具體做法。

## VI. Processes 🟢

- Worker 無狀態；session、資料都在 D1。
- Durable Object 是有狀態的，但持久狀態都在 D1 或 DO storage；記憶體只是快取：
  - presence 存在 WebSocket attachment，喚醒時重建。
  - spin 狀態存在 DO storage，計時用 alarm。

## VII. Port binding 🔵

Workers 不自己綁定 port，由平台處理。對應的精神「app 自成一體，不依賴外部注入的 web server」是符合的：一個 Worker 同時提供 SPA、API、WebSocket。

## VIII. Concurrency 🔵

Worker 由平台自動水平擴展；即時房間以 Event 為單位切分成獨立的 DO。

## IX. Disposability 🟢

- Worker 冷啟動極快。
- DO 隨時可能被回收：狀態不依賴記憶體（見 VI）；client 斷線後以指數退避重連，連上後由 `room.snapshot` 補齊。

## X. Dev/prod parity 🟡

**符合的部分**
- 本機 `wrangler dev` 使用與正式環境相同的 workerd runtime，D1 也是 SQLite。
- staging 與 production 結構完全相同（各自的 Worker、D1、Discord Application）。
- merge 到 `main` 即部署到 staging，時間差小。

**妥協**
- 本機以 `DiscordSDKMock` 模擬 Activity；真實的 Discord proxy、CSP、授權流程只在 staging 驗證。

## XI. Logs 🟢

- 一律輸出結構化 JSON 到 stdout（`console.log`），由 Workers Logs 收集。
- 不記錄 token、cookie、破冰回答內容。
- 之後可接 Logpush，不需修改程式。

## XII. Admin processes 🟢

- D1 migration 由 CI 以 `wrangler d1 migrations apply` 執行。
- 一次性管理工作寫成 `apps/worker/scripts/*` 或 admin API，與主程式共用程式碼與 schema；不在 D1 console 手動下 SQL（緊急狀況例外，事後補成 script）。
- 使用者角色管理透過 admin 管理頁。
