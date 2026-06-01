# 系統流程圖與路由對照文件 (FLOWCHART) - ZenFocus

本文件詳細規劃了 **ZenFocus** 的使用者操作流程 (User Flow)、系統內部交互序列圖 (Sequence Diagrams) 以及完整的功能與路由對照表。

---

## 1. 使用者流程圖 (User Flow)

使用者進入 ZenFocus 網頁後的完整操作路徑如下圖所示。系統分為三大主模組：**番茄鐘計時器**、**多軌白噪音混音器**、與**待辦任務看板**，並包含計時完成後的**心流日誌與數據統計**。

```mermaid
flowchart TD
    Start([使用者開啟網頁]) --> Home[首頁 - ZenFocus 主控制台]
    
    %% 主功能分流
    Home --> Choice{選擇操作模組}

    %% 1. 番茄鐘模組
    Choice -->|1. 計時器控制| Timer[番茄鐘控制面板]
    Timer --> TimerAction{要執行什麼？}
    TimerAction -->|設定時長/切換模式| Mode[切換 專注 / 短暫休息 / 長休息] --> PlayTimer
    TimerAction -->|開始 / 暫停| PlayTimer{計時器倒數中}
    PlayTimer -->|倒數結束| SingingBowl[播放禪意磬聲]
    SingingBowl --> Popup[彈出「情緒溫度計」自評視窗]
    Popup --> FillJournal[評估專注度 + 選擇當前情緒 + 寫下筆記]
    FillJournal --> SaveJournal[點擊提交日誌] --> Home

    %% 2. 混音器模組
    Choice -->|2. 白噪音混音| Mixer[多軌混音器]
    Mixer --> MixerAction{調整背景音效}
    MixerAction -->|開啟/關閉單軌| ToggleTrack[切換大自然音軌: 雨聲/篝火/環境鋼琴等] --> MixerOutput
    MixerAction -->|拉動滑桿| SlideVol[微調各音軌音量比例] --> MixerOutput
    MixerAction -->|特殊功能| SpecialMix[點擊 Surprise Me 隨機混音 / Mute All 靜音] --> MixerOutput
    MixerOutput[輸出個人專屬專注音場] --> Home

    %% 3. 任務看板模組
    Choice -->|3. 任務看板| Todo[待辦任務看板]
    Todo --> TodoAction{任務操作}
    TodoAction -->|新增| AddTask[輸入任務名稱、優先級、預估番茄鐘 🍅] --> SaveTask[存入待辦清單]
    TodoAction -->|專注鎖定| LockTask[點擊鎖定任務] --> PinTask[將任務文字懸浮置頂於番茄鐘正下方]
    TodoAction -->|打勾完成| DoneTask[標記任務為已完成] --> Confetti[觸發五彩紙屑 Confetti 動畫]
    TodoAction -->|刪除| DelTask[從清單中移除任務]
    SaveTask & PinTask & Confetti & DelTask --> Home

    %% 4. 統計儀表板模組
    Choice -->|4. 查看成效| Dashboard[進入數據統計頁面]
    Dashboard --> ViewStats[查看當日/當週累積專注時間]
    Dashboard --> ViewCharts[檢視專注趨勢折線圖與心流圓餅圖]
    Dashboard --> ViewTimeline[瀏覽歷史心流日誌時間軸]
    ViewStats & ViewCharts & ViewTimeline --> ReturnHome[點擊返回主控制台] --> Home
```

---

## 2. 系統序列圖 (Sequence Diagrams)

### 2.1 新增任務並渲染至看板
本序列圖描述使用者在前端新增待辦任務，系統將資料寫入 SQLite 資料庫並即時局部更新看板的流程：

```mermaid
sequenceDiagram
    autonumber
    actor User as 使用者
    participant Browser as 瀏覽器 (static/js/main.js)
    participant Flask as Flask 路由 (routes/tasks_api.py)
    participant Model as Task 模型 (models/task.py)
    participant DB as SQLite3 資料庫

    User->>Browser: 在輸入框填寫任務、優先級與預估番茄數 🍅
    User->>Browser: 點擊「新增任務」按鈕
    Browser->>Flask: POST /api/tasks (傳送 JSON 數據)
    Note over Flask: 驗證欄位是否填寫正確
    Flask->>Model: 呼叫 Task.create(title, priority, estimated_tomatoes)
    Model->>DB: 執行 SQL: INSERT INTO tasks (...)
    DB-->>Model: 返回插入成功與新建的 Task ID
    Model-->>Flask: 返回封裝好的 Task 物件
    Flask-->>Browser: 回傳 JSON: {status: "success", task: {...}}
    Note over Browser: 動態渲染 HTML 元素並掛載至 DOM<br/>(保持背景白噪音與計時器不中斷)
    Browser-->>User: 畫面即時出現新任務
```

---

### 2.2 番茄鐘結束、提交情緒日誌與更新統計
本序列圖描述番茄鐘結束後，使用者填寫專注評估心情，後端進行數據持久化，並即時更新統計看板的流程：

```mermaid
sequenceDiagram
    autonumber
    actor User as 使用者
    participant Browser as 瀏覽器 (static/js/timer.js)
    participant Flask as Flask 路由 (routes/journal_api.py)
    participant Model as Journal 模型 (models/journal.py)
    participant DB as SQLite3 資料庫

    Note over Browser: 計時倒數歸零 (00:00)
    Browser->>User: 播放 Singing Bowl 磬聲，彈出評估 Modal 視窗
    User->>Browser: 填寫專注效率(流暢)、心情(平靜)、並寫下心得筆記
    User->>Browser: 點擊「儲存心流日誌」
    Browser->>Flask: POST /api/journal (傳送 JSON 數據)
    Flask->>Model: 呼叫 MoodJournal.create(mood, efficiency, note, focus_duration)
    Model->>DB: 執行 SQL: INSERT INTO mood_journals (...)<br/>同時寫入或更新 focus_logs 專注總時間
    DB-->>Model: 返回儲存成功
    Model-->>Flask: 返回儲存成功狀態
    Flask-->>Browser: 回傳 JSON: {status: "success"}
    Note over Browser: 觸發 Confetti 特效，關閉 Modal
    Browser-->>User: 系統提示日誌已保存，使用者點選統計頁可看到更新後的心流曲線
```

---

## 3. 功能與路由對照表 (Routing & Page Mapping)

ZenFocus 的所有頁面路由與 RESTful API 設計規劃如下：

### 3.1 網頁頁面路由 (Page Routes)
負責直接渲染 HTML 模板：

| 功能名稱 | URL 路徑 | HTTP 方法 | Jinja2 模板檔案 | 說明 |
| :--- | :--- | :--- | :--- | :--- |
| **首頁控制台** | `/` | `GET` | `index.html` | 系統主介面。加載番茄鐘計時器、多軌混音器與今日待辦任務清單。 |
| **數據統計儀表板** | `/dashboard` | `GET` | `dashboard.html` | 展示專注時間統計圖表、情緒比例分析與歷史心流日誌時間軸。 |

---

### 3.2 RESTful API 路由 (JSON API Routes)
負責處理前端 JavaScript 的非同步請求 (Fetch API)，不刷新網頁，以保證白噪音音軌播放的連續性：

| 模組分類 | API 端點 | HTTP 方法 | 請求參數 (JSON) | 響應結果 (JSON) | 說明 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **任務看板** | `/api/tasks` | `GET` | 無 | `[ { "id": 1, "title": "...", "priority": "high", "completed": false }, ... ]` | 獲取所有未完成與已完成的任務清單。 |
| | `/api/tasks` | `POST` | `{ "title": "寫作業", "priority": "high", "estimated_tomatoes": 3 }` | `{ "status": "success", "task": { "id": 12, ... } }` | 新增一項任務。 |
| | `/api/tasks/<int:id>/complete` | `POST` | 無 | `{ "status": "success" }` | 標記特定任務為已完成（打勾）。 |
| | `/api/tasks/<int:id>/lock` | `POST` | `{ "locked": true }` | `{ "status": "success" }` | 將特定任務鎖定置頂（懸浮於計時器下方）。 |
| | `/api/tasks/<int:id>` | `DELETE` | 無 | `{ "status": "success" }` | 刪除特定任務。 |
| **專注統計** | `/api/focus/log` | `POST` | `{ "duration": 1500, "mode": "focus" }` | `{ "status": "success", "log_id": 45 }` | 當番茄鐘完成時，向後端紀錄一筆專注/休息時間紀錄。 |
| **心流日誌** | `/api/journal` | `POST` | `{ "mood": "平靜", "efficiency": "極高", "note": "完成第3章開發！" }` | `{ "status": "success" }` | 儲存專注完成後的情緒日誌，寫入資料庫。 |
| | `/api/journal` | `GET` | 無 | `[ { "id": 1, "created_at": "...", "mood": "平靜", "note": "..." }, ... ]` | 獲取所有歷史情緒日誌，用於渲染統計頁的時間軸。 |
