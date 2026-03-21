# RPJQ 2 — 維護文檔（Maintenance Guide）

> 給其他 AI 或開發者看的文件。記錄此專案的架構、部署流程、修改注意事項。

---

## 專案概述

**RPJQ 小工具** 是一個 Romeo & Juliet Party Quest（RJPQ）的隊伍協調工具。

- **功能**：多人房間、顏色選擇、10 層 × 4 門格子的即時同步標記板
- **技術**：純 HTML + CSS + Vanilla JS（無框架），透過 MQTT 做即時同步
- **MQTT Broker**：HiveMQ 公開 Broker（`wss://broker.hivemq.com:8884/mqtt`），無需帳號
- **部署**：GitHub Pages（靜態網站）

---

## 資料夾結構

```
D:\game\RPJQ 2\          ← 本地工作目錄 / Git 根目錄
├── docs/
│   ├── index.html       ← ★ 主程式（唯一需要改的檔案）
│   └── find.jpg         ← 靜態圖片資源
├── index.html           ← 根目錄備份（可與 docs/ 保持一致或忽略）
├── MAINTENANCE.md       ← 此文件
├── README.md
├── package.json
└── .git/
```

**GitHub Pages 設定**：
- Repository：`thumb168888/rpjq2-jack`
- Branch：`main`
- Folder：`/docs`
- 線上網址：https://thumb168888.github.io/rpjq2-jack/

---

## 程式架構（`docs/index.html`）

所有邏輯都在這一個 HTML 檔案裡，共約 1360 行。

### 主要狀態變數

| 變數 | 說明 |
|------|------|
| `client` | MQTT.js 客戶端實例 |
| `roomCode` | 6 位數房間號 |
| `roomPass` | 房間密碼 |
| `myId` | 本機唯一 ID（每次開頁面重新生成） |
| `myColor` | 本機選擇的顏色 |
| `myName` | 本機選擇的名字 |
| `isHost` | 是否為房間主機 |
| `hostId` | 房主的 myId |
| `hasState` | 是否已接收過完整房間狀態（`full_state`） |
| `playerMap` | `{ peerId: { color, name } }` 所有其他玩家 |
| `doors` | `Array[10][4]{ colors:[], wrongMark:bool }` 門格子狀態 |

### MQTT Topic 設計

```
rjpq-tool-v2/{roomCode}/info    ← retained, QoS 1：房間存在資訊／房主心跳
rjpq-tool-v2/{roomCode}/events  ← QoS 1：所有即時事件
```

### 主要 Event 類型

| type | 方向 | 說明 |
|------|------|------|
| `hello` | 任意→所有 | 玩家進房/重連時廣播自己 |
| `host_ping` | host→所有 | 每 10 秒心跳，確認房主在線 |
| `request_state` | 任意→所有 | 要求其他人傳送完整狀態 |
| `full_state` | 有資料者→發送者 | 傳完整房間快照 `{ doors, players }` |
| `door_toggle` | 任意→所有 | 點擊門格子，帶 `{ floor, door, color }` |
| `claim_name` | guest→host | 申請名字 |
| `claim_color` | guest→host | 申請顏色 |
| `player_update` | host→所有 | 確認玩家名字/顏色 |
| `claim_reject` | host→申請者 | 拒絕申請，帶 reason |
| `clear_all` | 任意→所有 | 清除所有門格子 |
| `leave` | 離開者→所有 | 正常離房 |
| `disconnect` | MQTT will | 斷線遺囑 |

### 同步機制

1. 玩家進房後發 `hello` + `request_state`
2. 收到 `request_state` 且 `hasState === true` 的玩家回傳 `full_state`
3. 收到 `full_state` 且 `hasState === false` 時，套用到本機並設 `hasState = true`
4. **⚠️ 關鍵**：從手機背景切回時，`attemptForegroundRecovery()` 會先把 `hasState = false`，再送 `request_state`，確保可以接收最新狀態

---

## 如何修改與推送

### 前置需求

- 已安裝 Git
- 已 clone 或有 `D:\game\RPJQ 2\` 資料夾
- 已設定 GitHub 帳號（可用 SSH 或 HTTPS）

### 修改流程

1. **編輯主程式**：只改 `docs/index.html`（根目錄的 `index.html` 僅為備份，可同步）
2. **測試**：直接用瀏覽器打開 `docs/index.html`（file:// 協議），或架本機 server
3. **推送**：

```powershell
# 在 D:\game\RPJQ 2\ 目錄執行
git add docs/index.html
git commit -m "你的說明"
git push origin main
```

4. **等 GitHub Pages 部署**：通常 30 秒 ~ 1 分鐘，到 https://github.com/thumb168888/rpjq2-jack/actions 確認部署狀態

---

## 已知限制與注意事項

### HiveMQ 公開 Broker

- **無認證**：任何人知道房間號和密碼就能加入
- **保留訊息（retained）**：`/info` topic 是 retained，房主離開時會清空（`client.publish(topic, '', {retain:true})`），確保舊房間不殘留
- **不保證房間完全私密**：這是輕量工具，非正式安全架構

### 同步設計

- **QoS 1**：所有發布都用 `qos:1`（至少送達一次），在不穩定網路下更可靠
- **`hasState` 旗標**：控制是否接受 `full_state` 更新，切回前景時會重設為 `false`
- **右鍵 wrongMark**：是本地私人記錄，**不同步給隊友**，刻意設計

### 已知問題

- 如果所有人同時斷線再重連，可能沒有人有 `hasState === true` 可以回傳狀態（需要重新標記）
- `roomEntered` 後重連時，guest 端有 `ROOM_STALE_MS`（10 分鐘）的寬限期，超時會提示重新加入

---

## 版本歷史

| 日期 | 說明 |
|------|------|
| 2026-03-21 | 修復手機/電腦不同步 bug：`hasState` 重設邏輯、QoS 0 升級為 QoS 1 |
| 2026-03-19 | 初始版本發布，MQTT 版（取代舊版 PeerJS） |
