# ADR 0002：獨立 Web App，Discord Activity 為入口之一；不做主動推播

- 狀態：已接受
- 日期：2026-10-03

## 背景

初始需求包含 Bot 頻道通知（US-08）與活動喚醒 Modal（US-04）。但 app 的設計是**不安裝在 guild**，所有 Discord 互動都由使用者觸發：

- 沒有 bot 在 guild 中，就不能主動在頻道發訊息，也不能私訊使用者。
- Interaction token 只有 15 分鐘有效，無法排程提醒。
- Activity 的 iframe 中無法註冊 Service Worker 或請求通知權限。

此外，產品未來可能不只用在 Activity 情境。

## 決策

- Huddle 是**獨立的 Web App**，網頁入口（Discord OAuth + cookie session）與 Activity 入口（Embedded App SDK + Bearer token）並列，都是 MVP 的一部分。
- 唯一的 slash command：`/huddle openwebpage`，回傳 ephemeral 的 webapp URL。
- **不做推播**。改為拉取式：
  - 進入 app 時查詢「進行中／即將開始」的場次，顯示 CTA。
  - 取消等事件寫入一次性 `notices`，進入 app 時顯示一次。

## 影響

- 不需要 Cron Trigger、Web Push、常駐 bot。
- 沒打開 app 的使用者不會收到任何提醒，這個風險已接受。
- 大部分功能可在一般瀏覽器開發與測試，只有 Activity 特有行為需要上 staging 驗證。
