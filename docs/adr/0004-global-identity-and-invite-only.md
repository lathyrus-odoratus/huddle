# ADR 0004：Global 身份、系統自管權限、邀請制加入

- 狀態：已接受
- 日期：2026-10-03

## 背景

Activity 可能在多個 guild 啟動，網頁入口則完全沒有 guild 的概念。如果以 Discord role 或 guild 判斷權限與可見性，會出現「在 A guild 看得到、在網頁看不到」這類不一致。

## 決策

- 使用者是 **global 身份**：`users` + `user_identities(provider, provider_user_id)`，目前只有 Discord，但預留其他登入方式。
- 權限由系統自行管理，**不使用 Discord role**：
  - 系統層：`admin` / `creator` / `user`。只有 creator 以上能建立 Series；初始 admin 由設定指定。
  - Series 層：`organizer` / `member`，存取控制只看 `series_members`。
- 加入 Series **只透過邀請**（連結或代碼），每個 Series 一組有效邀請，可重新產生；可開啟審核。不依 guild/channel 自動顯示或加入。
- 顯示名稱是單一的 `display_name`。建立帳號時以當下 guild 暱稱（或 `global_name`）作為預設值，之後由使用者自行修改，不再自動同步。

## 影響

- 不同入口的規則完全一致。
- Activity 授權需要多一個 `guilds.members.read` scope（取暱稱用）。
- 需要自建 admin 管理頁來授予 creator。
