# ADR 0007：一開始就建立 staging 與 production 兩套環境

- 狀態：已接受
- 日期：2026-10-03

## 背景

Discord Activity 的 URL Mapping、OAuth redirect、Interactions Endpoint 都綁在 Discord Application 上，因此要在 Discord 內驗證 Activity，需要一個 Discord 連得到的固定網址。可選擇用本機 tunnel，或部署到雲端的 staging。

## 決策

- 建立兩個 Discord Application：**Huddle (Staging)** 與 **Huddle**。
- Cloudflare Worker 兩個 environment，各自一個 D1：
  - staging：`huddle-staging.miao-bao.cc`，D1 `huddle-staging`
  - production：`huddle.miao-bao.cc`，D1 `huddle-prod`
- staging 用一層子網域，以符合 Universal SSL（`*.miao-bao.cc`）。
- **本機只負責 UI 修改的檢視**：瀏覽器 + HMR，Activity 行為以 `DiscordSDKMock` 模擬；OAuth 使用 staging Application 並允許 localhost redirect。
- 部署：merge 到 `main` 自動部署 staging；打 tag 或手動觸發部署 production。

## 影響

- 不需要維護 named tunnel；需要時才臨時開 quick tunnel。
- 在 Discord 內驗證改動需要先部署到 staging（約數十秒）。
- 機密的管理方式（`wrangler secret` 或 remote vault 注入）尚未決定。
