# ADR 0003：REST 負責寫入、DO 負責房間協調、WebSocket 只承載暫態

- 狀態：已接受
- 日期：2026-10-03

## 背景

最初考慮讓所有寫入都經由 WebSocket 進入 Durable Object，由 DO 序列化寫入並廣播。但很多寫入跟即時房間無關，例如活動前填寫出席意向、意願：使用者可能只是打開頁面填完就走，為此建立 WebSocket 連線並不合理。另一方面，隨機揭曉需要排他（不能同時觸發兩次）與計時（動畫結束才送出內容），這些適合由單執行緒的 DO 負責。

## 決策

- **權限檢查一律在 Worker**（Hono middleware），只寫一份。
- **單純的持久化寫入**（rsvp、簽到、破冰作答、意願、新增意願選項）：REST → Worker → D1，寫完呼叫 DO 的 `broadcast()`。
- **需要房間協調的動作**（開始、結束、手動/隨機揭曉、踢除連線）：REST → Worker 驗權限 → DO RPC，由 DO 寫 D1 並廣播。
- **WebSocket 的 client → server 只承載暫態互動**（表情反應，未來的點擊小互動）；presence 由連線本身表示。
- D1 是 source of truth；DO 的記憶體只放 presence 與 spin 等暫態。

## 影響

- 活動前、活動後、沒有人在房間時都能正常寫入。
- DO 保持精簡，不重複實作權限。
- 廣播失敗時，client 重連取得從 D1 重組的 snapshot，自然恢復一致。
- 揭曉的「內容在動畫結束前不可見」以 `revealed_at` 設為未來時間點實作，讀取一律以 `revealed_at <= now` 判斷。
