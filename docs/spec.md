# Huddle 技術規格（MVP）

> 狀態：規格定案，尚未實作
> 來源：[初始需求](./initial-requirements.md) 與後續討論；關鍵決策見 [ADR](./adr/)

---

## 1. 產品概述

Huddle 是一個**獨立的 Web App**，以 **Discord Activity 作為主要入口之一**，提供社群聚會的：

- 系列活動（Series）與場次（Event）管理
- 出席意向、簽到
- 破冰問題與「主持人控制的逐一揭曉」
- 下次聚會時間意願收集與主辦決議
- 即時多人互動（線上名單、簽到廣播、表情反應）
- 以 Series 為單位的歷史回顧

### 1.1 與初始需求的差異

| 初始需求 | 調整後 |
| --- | --- |
| US-04 活動喚醒 Modal | 改為**拉取式**：使用者進入 app 時查詢「進行中／即將開始」的場次，顯示 CTA。不做推播 |
| US-08 Bot 頻道通知 | **不做**。app 不安裝在 guild，不主動發訊息；改以 app 內一次性通知（notices）呈現取消等事件 |
| US-07 歷史會議記錄 | 會議記錄（筆記）**延後**；歷史回顧 = 簽到 + 破冰回答 + 意願/決議 |
| 意願類型 (a)/(b)/(c) 三選一 | 改為**固定組成的表單**：相對時間選項 + 絕對時段選項 + 自由填寫（見 §4.6） |
| 以 guild 為範圍 | **global 身份**，存取控制以 Series 成員資格為準，與 guild 無關 |

### 1.2 MVP 範圍

**包含**：Activity 與網頁雙入口登入、Series/Event 管理、邀請制加入、出席與簽到、破冰盲揭（手動與隨機動畫）、意願收集與鎖定、取消場次與一次性通知、即時線上名單與表情反應、個人狀態頁、Series 場次翻閱、admin 管理頁、`/huddle openwebpage` slash command、i18n（zh-TW、en）、深淺色主題、PiP 精簡畫面。

**不包含（延後）**：會議記錄、改期、Series 封存、帳號刪除、活動環節（phase）、投票、點擊小互動、「下一場已鎖定」通知、推播通知、E2E 測試。

---

## 2. 名詞

| 名詞 | 說明 |
| --- | --- |
| Series | 系列活動，例如「週三讀書會」。成員資格、權限、邀請都以 Series 為單位 |
| Event | Series 底下的一個場次，有編號 `seq`（#1、#2…） |
| 待辦 Event | 狀態為 `scheduled` 的 Event。每個 Series **最多一場** |
| 主辦 organizer | Series 層角色，管理 Series 設定、成員、場次排程 |
| 主持人 facilitator | Event 層角色，控制即時房間（揭曉破冰） |
| 成員 member | Series 層角色。「訂閱」= 是 Series 成員 |
| 意願 willingness | 對「下一場時間」的意見收集，掛在第 N 場，鎖定後產生第 N+1 場 |
| 房間 room | 一個 Event 的即時互動空間，由一個 Durable Object 負責 |
| notice | 一次性通知，使用者進入 app 時顯示一次 |

---

## 3. 角色與權限

### 3.1 三層角色

| 層級 | 角色 | 來源 |
| --- | --- | --- |
| 系統 | `admin` / `creator` / `user` | admin 由設定指定初始名單，admin 可在管理頁授予 `creator` |
| Series | `organizer` / `member` | 建立者自動成為 organizer；organizer 可升級成員為 organizer |
| Event | facilitator（0..n 人） | 建立 Event 時預設為建立者；organizer 可指派任何 Series 成員 |

- **organizer 隱含擁有該 Series 所有 Event 的主持權限**（主持人缺席時可接手）。
- 一個 Series 可有多位 organizer；**最後一位 organizer 不能退出**。
- 所有權限由系統自行管理，**不使用 Discord role**。

### 3.2 權限矩陣

| 動作 | admin | creator | organizer | facilitator | member |
| --- | :-: | :-: | :-: | :-: | :-: |
| 管理使用者系統角色 | ✅ | | | | |
| 建立 Series | ✅ | ✅ | | | |
| 修改 Series 設定、重新產生邀請、審核加入 | | | ✅ | | |
| 升降 organizer、移除成員 | | | ✅ | | |
| 建立 / 修改 Event、指派主持人 | | | ✅ | | |
| 取消 Event | | | ✅ | | |
| 鎖定下次時間 | | | ✅ | | |
| 新增絕對時段意願選項 | | | ✅ | ✅ | |
| 開始 / 結束 Event | | | ✅ | ✅ | |
| 揭曉破冰（手動 / 隨機） | | | ✅ | ✅ | |
| 修正他人簽到 | | | ✅ | ✅ | |
| 填出席意向、簽到、作答、填意願、送表情 | | | ✅ | ✅ | ✅ |
| 退出 Series | | | ✅¹ | | ✅ |

¹ 非最後一位 organizer 時。

> admin 不自動擁有 Series 內容權限；**任何角色都看不到未揭曉的破冰內容**。

---

## 4. 領域規則

### 4.1 Series 與成員

- 加入**只走邀請**：每個 Series 同時只有一組有效邀請（連結 `https://<domain>/join/<code>` 與代碼 `<code>`），organizer 可「重新產生」使舊碼失效。
- `join_requires_approval` 開啟時，加入後狀態為 `pending`，待 organizer 核准。
- Activity 內提供「輸入邀請碼」入口（Discord 內點連結會開瀏覽器，不會進 Activity）。
- 成員退出或被移除：**歷史資料保留**（簽到、破冰回答仍以其名稱顯示），但不能再進入該 Series。被移除時，Worker 通知該 Series 進行中房間將其連線踢除。

### 4.2 Event 生命週期

```
               ┌──開始(主持人/主辦)──▶ live ──結束(主持人/主辦)──▶ ended
 建立 ─▶ scheduled
               └──取消(主辦)──▶ cancelled
```

- 狀態**完全由人手動推進**，時間不會自動改變狀態。忘記開始/結束的風險已接受。
- Event 欄位只有預定的 `starts_at`，以及實際的 `started_at`、`ended_at`，沒有預定結束時間。
- **每個 Series 最多一場 `scheduled`**。已有待辦 Event 時，不能再建立或鎖定產生新的一場。
- **編號**：建立 Event 時從 `series.next_seq` 取號並寫入 `events.seq`，**取消的場次也佔號**。
- 只有 `scheduled` 可以取消；`live` 只能結束。取消可填原因。
- **結束**時若有未揭曉的破冰回答，前端提示主持人；確認後由房間一次全部揭曉，再切為 `ended`。不論如何，`ended` 之後所有回答都公開。

### 4.3 取消與下一場

- 取消由 organizer 執行；取消後 Series 沒有待辦 Event，**下一場由 organizer 手動建立**（不自動解除上一場的鎖定）。
- 取消時對 Series 所有 active 成員寫入一筆 `event_cancelled` notice（見 §4.9）。

### 4.4 出席與簽到

出席拆成「事前意向」與「實際簽到」兩個獨立欄位：

- `rsvp`：`null`（未回答）| `yes` | `maybe` | `no`。任何時候都能改，以查詢當下為準。
- `checked_in_at`：使用者在 Event `live` 時**手動按簽到**；主持人/主辦可替他人勾選或取消（`checked_in_by` 記錄操作者）。
- 簽到瞬間由房間廣播，所有人看到簽到特效。

顯示狀態（查詢時推導，不儲存）：

| 條件 | 顯示 |
| --- | --- |
| `checked_in_at` 有值 | 已簽到 |
| Event `ended` 且 `rsvp = yes` 且未簽到 | 預定參加但缺席 |
| Event `ended` 且 `rsvp = maybe` 且未簽到 | 未出席 |
| `rsvp = yes` / `maybe` / `no` | 預定參加 / 可能參加 / 不參加 |
| `rsvp = null` | 未回答 |

參與記錄「出席 X / Y 場」：X = 已簽到場次；Y = 使用者加入 Series 後 `ended` 的場次（不含 cancelled）。

### 4.5 破冰（盲揭）

- 每個 Event **最多一題**，選填。**還沒有人作答前**可以修改題目，有人作答後鎖定。
- 作答：Event 未結束前任何時候都可以作答（含進行中遲到的人）；**揭曉後鎖定**不能修改。
- **未揭曉的內容只有作者本人看得到**——主持人、主辦、admin 都看不到。其他人只看到「某某已作答」。這個過濾在 query 層執行，不依賴前端。
- 揭曉方式（主持人/主辦）：
  - **手動**：從已作答成員的頭像中點選一位（選的是人，不是內容）。
  - **隨機下一個**：server 從未揭曉者中隨機選一位，所有人同步看到轉盤動畫，動畫結束後才收到內容。
- 揭曉由房間（DO）序列化執行，同時只能有一個揭曉在進行中。
- `ended` 後所有回答公開。

### 4.6 下次時間意願與鎖定

意願表單由三部分組成：

| 部分 | 來源 | 時機 |
| --- | --- | --- |
| 相對時間選項（一週後、兩週後…） | Series 預設 `default_relative_options`，建立 Event 時複製為該 Event 的選項，可再修改 | Event 建立時 |
| 絕對時段選項（10/12 20:00…） | organizer 或 facilitator 臨時新增 | 隨時，通常在活動進行中 |
| 自由填寫 | 固定存在 | — |

- 成員可複選選項並另填一段文字；任何時候可修改，直到鎖定。
- 收集期間：從 Event 建立到 organizer 鎖定為止，與 Event 的 `live`/`ended` 狀態無關。
- 統計**公開**：誰勾了什麼所有成員都看得到。
- 新增選項時房間廣播，表單即時更新。
- **鎖定**（organizer）：手動輸入或確認下一場的 `starts_at`（就算從候選時段選，也要再確認一次），系統建立第 N+1 場 `scheduled` Event，並關閉第 N 場的意願表單。
  - 沿用：標題樣板（`{series.name} #{seq}`）、說明、相對時間選項、主持人（可改）。
  - 不沿用：破冰題目（每場重新出題）。
  - 已有待辦 Event 時不能鎖定。

### 4.7 首頁 CTA

使用者所屬（active）Series 中，符合以下條件的 Event **全部列出**：

```sql
status IN ('scheduled', 'live')
AND (status = 'live' OR now >= starts_at - series.cta_lead_minutes)
AND now < COALESCE(started_at, starts_at) + 12h
ORDER BY (status = 'live') DESC, starts_at ASC
```

點擊後導向 Event 頁（簽到、破冰、意願）。Event 的 `location` 由 Event 頁負責呈現。

### 4.8 Series 頁與歷史

- 預設顯示「最新場次」：有 `live` 的顯示 `live`，否則顯示 `seq` 最大的一場（含 cancelled，讓人看到取消原因）。
- 左右箭頭前後翻場次，下拉選單可直接跳到特定場次。
- 這就是 US-07 的歷史回顧入口（不另做跨 Series 時間軸）。

### 4.9 一次性通知（notices）

- 事件發生時對相關使用者各寫一筆（fan-out on write）。事件發生後才加入的成員不會收到。
- 進入 app 時查詢 `seen_at IS NULL` 的通知，回傳同時標為已讀——只通知一次，漏掉也沒關係。
- MVP 只有 `event_cancelled` 一種；表結構是通用的。

### 4.10 個人狀態頁（首頁）

由上而下：一次性通知 → CTA 列表 → 我的 Series 卡片（下一場時間、我的 rsvp）→ 參與記錄統計 → 我的歷史破冰回答（倒序）。不做跨 Series 的統一待辦。

### 4.11 顯示名稱

- `users.display_name` 是唯一的顯示名稱。
- 建立帳號時取預設值：Activity 從 guild 進入 → 該 guild 的暱稱；否則 → Discord `global_name`（再退回 `username`）。
- 使用者可自行修改；改過後 `display_name_source = 'custom'`。`auto` 狀態也**不會**在之後登入時自動同步（避免在不同 guild 間來回切換）。
- 頭像每次登入時從 Discord 同步。

---

## 5. 系統架構

```
                 ┌────────────── Cloudflare ───────────────────────────┐
Discord Activity │  Worker (Hono)                                       │
 (iframe, proxy) ─▶  ├─ Static Assets: Vue SPA                          │
                 │  ├─ /api/*        REST                               │──▶ Discord API
瀏覽器 / 網頁    ─▶  ├─ /auth/*       網頁 OAuth                          │   (OAuth、users/@me)
                 │  ├─ /ws           → Event Room DO（WebSocket）        │
Discord          │  └─ /interactions slash command                      │
 slash command  ─▶                                                       │
                 │  D1（source of truth）   Event Room DO × N            │
                 └─────────────────────────────────────────────────────┘
```

- **一個 Worker** 同時提供 SPA、API、WebSocket 與 interactions，一次 deploy 前後端一起上線。
- Activity 的 URL Mapping 只需一條：`/` → 該環境網域（HTTP 與 `wss` 都走這條）。

### 5.1 職責分工

> **權限檢查一律在 Worker；需要房間內協調（排他、計時、廣播順序）的動作交給 DO；單純的持久化寫入直接寫 D1，寫完通知 DO 廣播。WebSocket 的 client → server 只承載暫態互動。**

| 動作 | 路徑 |
| --- | --- |
| rsvp、意願、破冰作答、簽到、新增意願選項 | REST → Worker → D1 → `room.broadcast()` |
| 開始、結束、揭曉（手動/隨機）、踢除連線 | REST → Worker（驗權限）→ DO RPC → DO 寫 D1 並廣播 |
| 線上名單（presence） | WS 連線本身 |
| 表情反應（未來：點擊小互動） | WS client → DO → 廣播，不持久化 |

- DO 對應：**一個 Event 一個 DO**，`idFromName(eventId)`；使用 **WebSocket Hibernation API**，閒置不計費。
- 沒有人連線時呼叫 `broadcast()` 不會出錯，只是沒有對象。
- 一致性：廣播失敗時，client 重連取得的 `room.snapshot` 由 DO 從 D1 重組，自然補回。

### 5.2 隨機揭曉的流程

```
主持人 ─REST─▶ Worker（驗權限）─RPC─▶ EventRoom.spinRandom()
                                        ├─ 已有 spin 進行中 → 409
                                        ├─ 從「已作答且未揭曉」中隨機選一位
                                        ├─ D1: revealed_at = now + durationMs
                                        ├─ 廣播 icebreaker.spinning { candidates, winnerUserId, durationMs }
                                        └─ durationMs 後（DO alarm）廣播 icebreaker.revealed { userId, text }
```

- `revealed_at` 設為**動畫結束的時間點**，所有讀取都以 `revealed_at <= now` 判斷是否已揭曉，因此 REST 與 snapshot 不會在動畫結束前洩漏內容。
- 中途進房者的 snapshot 帶目前的 spin 狀態（winner、剩餘時間）。
- 動畫以「收到訊息時開始、持續 `durationMs`」播放，容許各端數百毫秒的誤差。
- 手動揭曉同樣經過 DO 以保持序列化，`durationMs` 為一段較短的揭曉動畫。

### 5.3 DO 被回收時的處理

使用 Hibernation 時，DO 閒置就會被移出記憶體，**記憶體中的變數隨時可能消失**。所以：

- **presence**：每條 WebSocket 用 `serializeAttachment()` 存 `userId`；喚醒時以 `ctx.getWebSockets()` 重建線上名單，不依賴記憶體中的 Map。
- **spin 狀態**：寫入 DO storage；計時一律用 **DO alarm**（會持久化，DO 被回收也會觸發），不使用 `setTimeout`。
- **client**：斷線後以指數退避重連（含 jitter），重新取得 ticket，連上後由 `room.snapshot` 補齊狀態。

---

## 6. 資料模型（D1）

時間一律存 **UTC epoch 毫秒（INTEGER）**；ID 使用 `crypto.randomUUID()`。以 Drizzle ORM 定義 schema，`drizzle-kit` 產生 migration，`wrangler d1 migrations apply` 套用。

```
users
  id PK, display_name, display_name_source ('auto'|'custom'),
  avatar_url, locale, system_role ('admin'|'creator'|'user'),
  created_at, updated_at

user_identities                       -- 預留多種登入方式
  provider ('discord'), provider_user_id, user_id FK,
  username, global_name, avatar, updated_at
  PK(provider, provider_user_id)

sessions
  token_hash PK, user_id FK, kind ('activity'|'web'), expires_at, created_at

series
  id PK, name, description, default_relative_options (JSON string[]),
  cta_lead_minutes (default 60), join_requires_approval (bool),
  invite_code UNIQUE, next_seq (default 1), created_by FK, created_at, updated_at

series_members
  series_id FK, user_id FK, role ('organizer'|'member'),
  status ('pending'|'active'|'left'|'removed'), joined_at, updated_at
  PK(series_id, user_id)

events
  id PK, series_id FK, seq, title, description, location,
  starts_at, status ('scheduled'|'live'|'ended'|'cancelled'),
  started_at, ended_at, cancelled_at, cancel_reason,
  icebreaker_question,
  willingness_locked_at, next_event_id FK,
  created_by FK, created_at, updated_at
  UNIQUE(series_id, seq)
  -- 「每個 Series 最多一場 scheduled」：partial unique index
  UNIQUE(series_id) WHERE status = 'scheduled'

event_facilitators
  event_id FK, user_id FK, PK(event_id, user_id)

event_attendance
  event_id FK, user_id FK, rsvp (null|'yes'|'maybe'|'no'), rsvp_updated_at,
  checked_in_at, checked_in_by FK
  PK(event_id, user_id)

icebreaker_answers
  event_id FK, user_id FK, text, revealed_at, created_at, updated_at
  PK(event_id, user_id)

willingness_options
  id PK, event_id FK, kind ('relative'|'absolute'),
  label, starts_at (absolute 時有值), sort_order, created_by FK, created_at

willingness_responses
  event_id FK, user_id FK, option_ids (JSON string[]), free_text, updated_at
  PK(event_id, user_id)

notices
  id PK, user_id FK, kind ('event_cancelled'), ref_id, payload (JSON),
  created_at, seen_at
  INDEX(user_id, seen_at)
```

> 相對時間選項在建立 Event 時「實體化」為 `willingness_options` 列，因此 Event 不需要獨立的 `relative_options` 欄位；「Event 覆寫」就是修改這些列。

---

## 7. REST API 輪廓

全部 JSON，請求與回應以 `packages/shared` 的 zod schema 定義，前端透過 Hono RPC client（`hc`）取得型別。

### 認證與個人

| Method | Path | 說明 |
| --- | --- | --- |
| GET | `/api/config` | 公開的執行時設定（`discordClientId`、`appUrl` 等），不需登入；見 §11.1 |
| POST | `/api/auth/activity` | `{ code }` → `{ discordAccessToken, sessionToken, user }` |
| GET | `/auth/discord/login` | 網頁 OAuth 起點（帶 `state`） |
| GET | `/auth/discord/callback` | 設定 session cookie 後導回 app |
| POST | `/api/auth/logout` | |
| GET | `/api/me` | 目前使用者 |
| PATCH | `/api/me` | `display_name`、`locale` |
| GET | `/api/me/home` | `{ notices, ctaEvents, seriesCards, stats, recentAnswers }`，回傳時把 notices 標為已讀 |

### Series、邀請、成員

| Method | Path | 權限 |
| --- | --- | --- |
| POST | `/api/series` | creator |
| GET / PATCH | `/api/series/:id` | member / organizer |
| GET | `/api/series/:id/events` | member（下拉選單用：id、seq、title、status、starts_at） |
| POST | `/api/series/:id/invite/regenerate` | organizer |
| GET | `/api/invites/:code` | 已登入（預覽 Series 名稱） |
| POST | `/api/invites/:code/join` | 已登入；限流 |
| GET | `/api/series/:id/members` | member |
| PATCH | `/api/series/:id/members/:userId` | organizer（升降角色、核准/拒絕 pending） |
| DELETE | `/api/series/:id/members/:userId` | organizer（移除）或本人（退出） |

### Event

| Method | Path | 權限 |
| --- | --- | --- |
| POST | `/api/series/:id/events` | organizer；已有待辦 Event 時 409 |
| GET | `/api/events/:id` | member |
| PATCH | `/api/events/:id` | organizer；`icebreaker_question` 有人作答後不可改 |
| PUT | `/api/events/:id/facilitators` | organizer |
| POST | `/api/events/:id/start` | facilitator → DO |
| POST | `/api/events/:id/end` | facilitator → DO（`{ revealAll: true }`） |
| POST | `/api/events/:id/cancel` | organizer；僅 `scheduled`；`{ reason }` |
| PUT | `/api/events/:id/rsvp` | member |
| POST | `/api/events/:id/check-in` | member（本人，僅 `live`） |
| PUT | `/api/events/:id/attendance/:userId` | facilitator（修正簽到） |
| PUT | `/api/events/:id/icebreaker/answer` | member（未揭曉前） |
| POST | `/api/events/:id/icebreaker/reveal` | facilitator → DO；`{ userId }` |
| POST | `/api/events/:id/icebreaker/reveal-random` | facilitator → DO |
| GET | `/api/events/:id/willingness` | member（選項、所有回覆、統計） |
| POST | `/api/events/:id/willingness/options` | facilitator（新增絕對時段）／organizer（全部） |
| PUT | `/api/events/:id/willingness/response` | member（未鎖定前） |
| POST | `/api/events/:id/willingness/lock` | organizer；`{ startsAt, title?, facilitatorIds? }` → 建立下一場 |
| POST | `/api/events/:id/ws-ticket` | member → `{ ticket }` |

### Admin

| Method | Path | 權限 |
| --- | --- | --- |
| GET | `/api/admin/users` | admin |
| PATCH | `/api/admin/users/:id` | admin（`system_role`） |

### 其他

| Method | Path | 說明 |
| --- | --- | --- |
| GET | `/ws?ticket=…` | WebSocket upgrade，轉交 Event Room DO |
| POST | `/interactions` | Discord interactions，驗證 Ed25519 簽章 |

---

## 8. WebSocket 協定

訊息以 `t` 為型別鍵，zod discriminated union 定義於 `packages/shared`。DO 依 `t` dispatch，每個 handler 一個模組，未來的 `phase.*`、`poll.*`、`fx.*` 只需新增 handler。

```ts
// client → server（只承載暫態互動）
{ t: "reaction", emoji: string }

// server → client
{ t: "room.snapshot", event, presence, attendance, answeredUserIds, revealedAnswers, spin, willingnessOptions }
{ t: "presence.join" | "presence.leave", userId }
{ t: "event.status", status, startedAt?, endedAt? }
{ t: "rsvp.changed", userId, rsvp }
{ t: "checkin.changed", userId, checkedInAt }        // 簽到特效
{ t: "icebreaker.answered", userId }                 // 不含內容
{ t: "icebreaker.spinning", candidates, winnerUserId, durationMs }
{ t: "icebreaker.revealed", userId, text }
{ t: "willingness.option_added", option }
{ t: "willingness.locked", nextEventId }
{ t: "reaction", userId, emoji }
{ t: "room.kicked", reason }                         // 被移出 Series
```

- 連線：`POST /api/events/:id/ws-ticket` 取得短效 ticket（約 30 秒，綁定 user 與 event 的簽章 token）→ `wss://<domain>/ws?ticket=…`。Worker 驗證 ticket 與成員資格後，帶 `userId` 轉給 DO。不把 sessionToken 放在 query string。
- 表情反應在 DO 內針對每位使用者節流（例如每秒 3 個）。
- 前端收到事件後直接更新 TanStack Query 的快取。
- 斷線重連策略見 §5.3。

---

## 9. 認證

### 9.1 Activity 入口

```
GET /api/config → discordClientId
  → new DiscordSDK(discordClientId)
SDK ready → sdk.commands.authorize({ scope: ['identify', 'guilds.members.read'] }) → code
  → POST /api/auth/activity { code, guildId? }
       Worker：用 client secret 換 Discord access token
             → GET /users/@me（含 locale）
             → 首次建立帳號且有 guildId：GET /users/@me/guilds/{guildId}/member 取暱稱
             → upsert users / user_identities → 建立 session
       ← { discordAccessToken, sessionToken, user }
  → sdk.commands.authenticate({ access_token: discordAccessToken })
  → 之後的 API 帶 Authorization: Bearer <sessionToken>
```

- 首次需同意授權，之後啟動靜默完成。
- `sessionToken` **只存在記憶體**，每次開 Activity 重新換取（iframe 的 storage 與 cookie 受第三方隔離影響，不依賴）。

### 9.2 網頁入口

- `/auth/discord/login` 導向 Discord OAuth（scope：`identify`，帶 `state` 防 CSRF）→ `/auth/discord/callback` → 設定 `httpOnly; Secure; SameSite=Lax` cookie。

### 9.3 Session

- 不透明隨機 token，D1 只存 hash；30 天滑動到期。
- 不使用 JWT：每個請求本來就要查 D1 取得角色與成員資格，且需要可立即撤銷。
- **不長期保存** Discord access token。

### 9.4 初始 admin

- 以設定值列出初始 admin 的 Discord user id；這些使用者登入時 `system_role` 設為 `admin`。

---

## 10. Slash command

- 只有一個：`/huddle openwebpage` → 回應 **ephemeral** 訊息，內容為該環境的 webapp URL。
- app 為 user-installable，不需要 bot 加入 guild。
- 不做排程訊息或主動推播。

---

## 11. 前端

| 項目 | 選擇 |
| --- | --- |
| 框架 | Vue 3 + Vite + TypeScript |
| 全域狀態 | Pinia（session、目前房間） |
| 資料抓取 | TanStack Query (Vue)；WS 事件直接更新快取 |
| API 型別 | Hono RPC client（`hc`） |
| 樣式 | Tailwind CSS v4 |
| 元件 | shadcn-vue（Reka UI），原始碼放進 repo 可完全客製 |
| 動畫 | motion-v（表情、簽到特效、揭曉轉盤） |
| i18n | vue-i18n；zh-TW、en。初始語系取 Discord `locale`，可在設定中改 |
| 主題 | 預設深色（與 Discord 一致），網頁版可切換淺色/深色 |

### 11.1 執行時設定

前端 build **不包含任何因環境而異的值**（不使用 `import.meta.env` 放環境設定），所有環境部署**同一份** `dist`。啟動時呼叫 `GET /api/config` 取得公開設定（例如 `discordClientId`、`appUrl`），由 Worker 從自己的環境變數組出。機密絕不出現在這個 endpoint。

### 11.2 Activity 特有處理

1. **路由**：依 URL 是否有 `frame_id` 判斷入口。Activity 用 `createMemoryHistory`（保留 SDK 需要的 query 參數），網頁用 `createWebHistory`。同一份 build。
2. **外部資源**：Discord 頭像 `cdn.discordapp.com` 先實測能否在 Activity 內直接載入；不行則加 URL Mapping 並在啟動時呼叫 `patchUrlMappings`。
3. **外部連結**：`<ExternalLink>` 元件，Activity 內使用 `sdk.commands.openExternalLink()`，網頁用一般 `<a>`。
4. **版面**：mobile-first 響應式；訂閱 SDK 的 layout mode，**PiP 模式顯示精簡畫面**（線上人數、簽到按鈕、表情列）。
5. **安全區域**：`env(safe-area-inset-*)`。
6. **本機開發**：Activity 專屬行為以 SDK 的 `DiscordSDKMock` 模擬。

---

## 12. 安全與限流

| 面向 | 做法 |
| --- | --- |
| 權限 | Hono middleware：`requireAuth` → `requireSeriesRole()` → `requireFacilitator()`；admin 路由 `requireSystemRole('admin')` |
| 輸入驗證 | 所有 REST body 與 WS 訊息以 zod 驗證；長度上限：破冰回答 500、意願自由填寫 200、名稱 50 字 |
| 盲揭 | query 層過濾 `revealed_at <= now` 或作者本人，不依賴前端 |
| 邀請碼 | 10 碼 base62；`/join` 限流 |
| 限流 | Workers Rate Limiting binding：登入、加入、ws-ticket；表情在 DO 內節流 |
| CSRF | 網頁 cookie `SameSite=Lax`，所有寫入檢查 `Origin`；Activity 用 Bearer 不受影響 |
| XSS | 依賴 Vue 預設跳脫；`location` 只將 `http(s)://` 轉為連結；不使用 `v-html` |
| WebSocket | 連線時檢查成員資格；移除成員時踢除連線 |
| Interactions | Ed25519 簽章驗證，失敗回 401 |
| 公開 repo | 機密不進版控；本機 `.dev.vars` 列入 `.gitignore` |

---

## 13. 環境與部署

| | 本機 | staging | prod |
| --- | --- | --- | --- |
| 用途 | UI 修改檢視（瀏覽器 + HMR + `DiscordSDKMock`） | 驗證真實 Activity | 正式環境 |
| 網域 | `localhost` | `huddle-staging.miao-bao.cc` | `huddle.miao-bao.cc` |
| Cloudflare | `wrangler dev`（miniflare 模擬 D1/DO） | Worker env `staging` | Worker env `production` |
| D1 | 本機 SQLite | `huddle-staging` | `huddle-prod` |
| Discord Application | Huddle (Staging) 的 OAuth（redirect 到 localhost） | Huddle (Staging) | Huddle |
| 部署 | — | merge 到 `main` 自動部署 | 打 tag 或手動觸發 workflow |

- staging 用 `huddle-staging`（一層子網域），以符合 Cloudflare Universal SSL（`*.miao-bao.cc`）。
- 每個 Discord Application 設定：Activity URL Mapping `/` → 環境網域；OAuth redirect `https://<網域>/auth/discord/callback`（staging 另加 localhost）；Interactions Endpoint `https://<網域>/interactions`。
- 本機也可用 `pnpm deploy:staging` 在 merge 前先推上 staging。
- 密集除錯 Activity 時可臨時開 cloudflared quick tunnel。

### 13.1 Build、Release、Run

- **每個 commit 只 build 一次**：CI 產出 Worker bundle 與 web `dist`，存成 artifact。
- staging 與 production **部署同一份 artifact**；production **只能部署已在 staging 部署過的 commit**。
- 每次 deploy 產生不可變的 Worker version ID，可以用 `wrangler rollback` 回退。
- 執行時設定由各環境自己提供（見 §11.1、§13.3），不寫進 artifact。

### 13.2 CI/CD（GitHub Actions）

- PR：typecheck、lint、test。
- `main`：build → 上傳 artifact → `wrangler d1 migrations apply --env staging` → 部署到 staging。
- tag `v*` 或手動觸發：取出**同一個 commit** 的 artifact → `migrations apply --env production` → 部署到 production。
- 使用 GitHub Environments（`staging`、`production`）分開存放各環境的機密；production 可設定需要人工核准。
- CI 使用最小權限的 Cloudflare API Token（Workers Scripts、D1、`miao-bao.cc` 的 Workers Routes）。

**Migration 規則（expand / contract）**：migration 在 deploy 之前套用，rollback 時也會出現「舊程式碼面對新 schema」的狀況，所以**任何單一 migration 都不能讓前一版程式碼壞掉**。要刪除或改名欄位時分兩次 release：先新增並讓程式碼改用新欄位，下一次 release 才刪除舊欄位。

### 13.3 設定與機密

- **`wrangler.jsonc`**：只放基礎設施拓撲（binding 名稱、D1 綁定、route 與網域），不放機密，也盡量不放會因環境改變的應用設定。
- **應用設定與機密**：一律從環境注入到 Worker 的環境變數 / secret。

| 名稱（暫定） | 類型 | 說明 |
| --- | --- | --- |
| `DISCORD_CLIENT_ID` | 設定（公開） | 也經 `/api/config` 提供給前端 |
| `DISCORD_CLIENT_SECRET` | 機密 | OAuth code 交換 |
| `DISCORD_PUBLIC_KEY` | 設定（公開） | 驗證 interactions 簽章 |
| `APP_URL` | 設定 | 該環境的網址，用於 OAuth redirect 與 slash command 回應 |
| `ADMIN_DISCORD_IDS` | 設定 | 初始 admin 名單 |
| `WS_TICKET_SECRET` | 機密 | WS ticket 簽章金鑰 |
| `CLOUDFLARE_API_TOKEN` | 機密（僅 CI） | 部署用 |

> **待定**：機密的來源與注入方式（GitHub Environments + `wrangler secret`，或採用外部 secret manager）。

---

## 14. 工程

```
huddle/
├─ apps/web          Vue SPA
├─ apps/worker       Hono API、EventRoom DO、/interactions、D1 migrations
├─ packages/shared   zod schema（REST 與 WS 協定）、型別、常數
└─ docs/             規格、ADR、初始需求
```

- pnpm workspace；Worker 以 Static Assets 帶出 `apps/web/dist`。
- 測試：
  - **Worker 為重點**：`@cloudflare/vitest-pool-workers`，在真實 D1/DO runtime 測權限、狀態轉換（取消、鎖定、取號、最多一場待辦）、盲揭過濾、DO spin 排他與廣播。
  - 前端：Vitest 測 composable 與 store 邏輯，不追求覆蓋率。
  - E2E：MVP 不做。
- Lint/format：ESLint flat config + Prettier。
- 觀測：Workers Logs；MVP 不接 Sentry。
- **Log 格式**：一律輸出結構化 JSON，例如 `{ level, msg, requestId, userId, eventId, ... }`，方便在 Workers Logs 依欄位查詢，之後接 Logpush 也不用改程式。不記錄 token、cookie、破冰回答內容。
- **一次性管理工作**：補資料、修正狀態這類工作，寫成 `apps/worker/scripts/*` 或 admin API，跟主程式**使用同一份程式碼與 schema**，針對指定環境執行。不在 D1 console 手動下 SQL；緊急狀況例外，但事後要補成 script。
- 依賴：`wrangler` 等工具都列為 devDependency 並鎖版本，CI 不使用全域安裝。

---

## 15. 未決事項

| 項目 | 處理時機 |
| --- | --- |
| 機密的來源與注入方式（GitHub Environments 或外部 secret manager） | 實作前 |
| 預先 build 的 Worker bundle 如何部署到不同環境（`--no-bundle` 等做法） | 建立 CI 時驗證 |
| `cdn.discordapp.com` 在 Activity 內能否直接載入 | 開發初期 spike |
| 揭曉動畫 `durationMs` 數值、表情節流參數 | 實作時調整 |

## 16. 延後功能（MVP 之後）

會議記錄（共筆）、改期、Series 封存、帳號刪除、活動環節（phase）、投票（是非、m 選 n）、點擊小互動、「下一場已鎖定」通知、推播通知、E2E 測試、Discord 以外的登入方式。
