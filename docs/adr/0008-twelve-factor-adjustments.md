# ADR 0008：依 12-Factor 調整設定、建置與執行方式

- 狀態：已接受
- 日期：2026-10-03

## 背景

用 [12-Factor App](https://12factor.net/) 檢查既有規劃。Codebase、Dependencies、Backing services、Concurrency、Dev/prod parity 已經符合；Port binding 由 Workers 平台處理。以下幾項有缺口：

- **III Config / V Build-Release-Run**：Activity 前端需要 Discord client ID。若在 build 時以 `import.meta.env` 寫入，staging 與 prod 會是兩份不同的產物；CI 也是每個環境各自重新 build。
- **VI Processes / IX Disposability**：使用 Hibernation 的 DO 隨時會被移出記憶體，記憶體中的暫態會消失。
- **V**：migration 在 deploy 前套用，rollback 時舊程式碼會面對新 schema。
- **XI Logs**：沒有規定 log 格式。
- **XII Admin processes**：沒有規範一次性的資料修正工作。

## 決策

- **A. 執行時設定**：前端 build 不含環境設定，啟動時從 `GET /api/config` 取得公開設定。`wrangler.jsonc` 只放基礎設施拓撲；應用設定與機密從環境注入。
- **B. Build once, deploy many**：每個 commit 只 build 一次並存成 artifact；staging 與 prod 部署同一份；prod 只能部署已在 staging 部署過的 commit。以 Worker version ID 與 `wrangler rollback` 作為不可變的 release。
- **C. Migration 採 expand / contract**：任何單一 migration 都不能讓前一版程式碼壞掉。
- **D. DO 可被回收**：presence 存在 WebSocket attachment；spin 狀態存在 DO storage，計時用 alarm；client 以指數退避重連，連上後用 snapshot 補齊狀態。
- **E. 結構化 JSON log**，不記錄 token 與破冰內容。
- **F. 一次性管理工作**寫成 repo 內的 script 或 admin API，與主程式共用程式碼與 schema。

## 影響

- 前端啟動時多一個請求（`/api/config`），可以跟登入流程並行或由 Worker 注入 HTML 來省掉。
- CI 需要處理 artifact 的保存與跨 workflow 取用，並驗證預先 build 的 bundle 如何部署到不同環境。
- 刪除或改名欄位需要兩次 release。
- 機密的來源（GitHub Environments 或外部 secret manager）仍待決定。
