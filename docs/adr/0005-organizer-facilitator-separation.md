# ADR 0005：主辦（organizer）與主持人（facilitator）分離

- 狀態：已接受
- 日期：2026-10-03

## 背景

社群聚會常見「系列由固定的人管理，但每場輪流主持」。管理 Series（成員、排程、設定）與控制單場即時房間（揭曉破冰，未來還有環節推進、投票）是不同範圍的權限。

## 決策

- **organizer** 是 Series 層角色：Series 設定、邀請與審核、成員管理、建立/修改/取消 Event、鎖定下次時間、指派主持人。可有多位，最後一位不能退出。
- **facilitator** 是 Event 層角色（`event_facilitators`，0..n 人）：開始/結束、揭曉破冰、新增絕對時段意願選項、修正簽到。預設為 Event 建立者，可指派任何 Series 成員。
- organizer **隱含擁有**該 Series 所有 Event 的主持權限，主持人缺席時可接手。
- 系統層的建立權限角色命名為 `creator`，避免與 Series 層的 organizer 撞名。

## 影響

- 權限檢查分成兩層 middleware：`requireSeriesRole()` 與 `requireFacilitator()`（後者也接受 organizer）。
