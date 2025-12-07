# 🧪 書籍搜尋系統 - 測試計畫 (Book Search Test Plan)

## 1. 測試目標 (Test Objectives)
本文件定義了針對 `Book-Search-Service` 的測試策略與案例，旨在驗證：
* **搜尋準確度 (Accuracy):** 確保買家能透過書名、課程名稱、老師姓名等多維度關鍵字找到目標書籍。
* **排序邏輯 (Ranking):** 驗證加權演算法是否正確（如：書名匹配的優先級高於課程匹配）。
* **效能 (Performance):** 驗證搜尋請求能否在 **500ms** 內完成 (NFR-BS-P-001)。
* **安全性 (Security):** 防止惡意查詢注入 (Query Injection) 並確保搜尋結果不包含敏感個資。

---

## 2. 測試環境 (Test Environment)
* **Environment:** Staging (準生產環境)
* **Search Engine:** Elasticsearch Cluster (v8+, with IK Analyzer)
* **Database:** PostgreSQL (Primary DB)
* **Tools:**
    * **Postman:** API 功能測試。
    * **JMeter / k6:** 效能與壓力測試。

---

## 3. 測試案例 (Test Cases)

### 3.1 功能性測試 (Functional Testing)

| 測試編號 | 測試名稱 | 前置條件 | 測試步驟 (Test Steps) | 預期結果 (Expected Result) |
| :--- | :--- | :--- | :--- | :--- |
| **TC-BS-001** | **關鍵字模糊搜尋** | 索引中存在書名為「計算機概論」的書籍。 | 1. 呼叫 `GET /search?q=計算機`。<br>2. 呼叫 `GET /search?q=計概` (模糊/同義詞)。 | 回傳結果中必須包含該書籍，且 `title` 欄位有高亮 (Highlight) 標籤。 |
| **TC-BS-002** | **多維度搜尋 (課程/老師)** | 書籍 A 對應課程「演算法」，老師「林某某」。 | 1. 呼叫 `GET /search?q=林某某`。<br>2. 呼叫 `GET /search?q=演算法`。 | 即使書名不含關鍵字，結果中仍必須出現書籍 A。 |
| **TC-BS-003** | **加權排序驗證 (Ranking)** | 存在兩本書：<br>A: 書名「Java程式設計」<br>B: 書名「數學」，描述含「Java」。 | 1. 呼叫 `GET /search?q=Java`。<br>2. 觀察回傳列表順序。 | 書籍 A (書名匹配) 的 `_score` 必須高於書籍 B (描述匹配)，因此 A 排在 B 前面。 |
| **TC-BS-004** | **狀態過濾 (已售出隱藏)** | 書籍 C 狀態為 `SOLD`，書籍 D 為 `AVAILABLE`。 | 1. 呼叫 `GET /search` (無關鍵字，列出所有)。 | 結果中**只能**看到書籍 D，書籍 C 必須被自動過濾掉 (AC-BS-004)。 |
| **TC-BS-005** | **進階篩選 (Filter)** | 存在不同價格與新舊的書籍。 | 1. 呼叫 `GET /search?price_max=500&condition=NEW`。 | 回傳結果的所有書籍價格皆 <= 500 且狀況為全新。 |

### 3.2 安全性測試 (Security Testing)

| 測試編號 | 測試名稱 | 攻擊手法 / 測試輸入 | 預期結果 (Expected Result) |
| :--- | :--- | :--- | :--- |
| **TC-SEC-001** | **Elasticsearch 注入嘗試** | 輸入特殊字符如 `OR 1=1`, `*`, `?`, `{}` 或 JSON 語法嘗試操控查詢 DSL。 | API 應對輸入進行 **Sanitization** (清洗)，將特殊字符視為普通文字或移除，回傳 `200 OK` (空結果) 或 `400 Bad Request`，且不洩露系統內部錯誤。 |
| **TC-SEC-002** | **敏感個資洩漏檢查** | 1. 執行任意搜尋請求。<br>2. 檢查 Response JSON 的 `hits` 內容。 | 每個項目的 JSON 中**不可包含**賣家的 `real_name`, `phone`, `email`。僅允許顯示非識別化的 `seller_id` 或公開暱稱。 |

### 3.3 效能測試 (Performance Testing)

針對 **NFR-BS-P-001** (速度) 與 **NFR-BS-E-001** (擴展性) 進行驗證。

| 測試編號 | 測試名稱 | 測試條件 | 預期結果 (Expected Result) |
| :--- | :--- | :--- | :--- |
| **TC-PERF-001** | **單次搜尋響應時間** | 資料庫有 10,000 筆書籍資料，執行複雜關鍵字查詢。 | API 響應時間 (Response Time) 應 **< 500ms**。 |
| **TC-PERF-002** | **高併發搜尋壓力測試** | 模擬 **100 個併發使用者 (CCU)** 持續執行搜尋與翻頁操作，持續 10 分鐘。 | 1. 錯誤率 < 1%。<br>2. 平均響應時間 < 800ms。<br>3. Elasticsearch CPU 使用率不超過 80%。 |

### 3.4 邊緣案例與異常測試 (Edge Case Testing)

驗證系統在極端或非預期輸入下的穩定性與錯誤處理能力。

| 測試編號 | 測試名稱 | 測試輸入 / 場景 | 預期結果 (Expected Result) |
| :--- | :--- | :--- | :--- |
| **TC-EDGE-001** | **極長關鍵字輸入** | 輸入超過 255 個字元甚至 10KB 的超長查詢字串。 | 系統應攔截並回傳 `400 Bad Request` (提示長度過長) 或自動截斷，**不可導致後端崩潰或記憶體溢出**。 |
| **TC-EDGE-002** | **特殊符號與 Emoji** | 搜尋包含 `📚`, `🔥`, `\n`, `\t` 或全形符號的關鍵字。 | 系統應能正常處理（忽略或作為普通字符），不應出現 500 錯誤。 |
| **TC-EDGE-003** | **空白查詢** | 僅輸入空白鍵或 URL encoded 的空白 (`%20`)。 | 視同無關鍵字，回傳所有書籍（或提示輸入關鍵字），不應報錯。 |
| **TC-EDGE-004** | **無效的分頁參數** | 呼叫 `GET /search?page=-1` 或 `page=9999999`。 | page=-1 應回傳 `400`；page=999999 應回傳 `200 OK` 但 `hits` 為空列表。 |

### 3.5 資料同步延遲測試 (Data Sync Latency Testing)

針對 **CQRS 架構** 的關鍵測試，驗證主資料庫的變更需要多久才會反映在搜尋結果中 (Eventual Consistency)。

| 測試編號 | 測試名稱 | 測試步驟 | 預期結果 (Expected Result) |
| :--- | :--- | :--- | :--- |
| **TC-SYNC-001** | **新增書籍同步時間** | 1. 呼叫刊登 API `POST /books` 新增書籍 X。<br>2. 立即（每 100ms）輪詢搜尋 API `GET /search?q=X`。<br>3. 記錄直到搜尋到該書所需的時間。 | 同步延遲 (Lag) 應 **< 2 秒** (視 Sync Worker 設定而定)。 |
| **TC-SYNC-002** | **售出狀態同步** | 1. 將書籍 Y 狀態更新為 `SOLD`。<br>2. 立即搜尋書籍 Y (不帶過濾)。<br>3. 記錄直到書籍 Y 從結果中消失（或狀態變更）的時間。 | 狀態變更應在 **< 2 秒** 內反映，避免買家看到無效庫存。 |

### 3.6 斷詞與同義詞測試 (Tokenization & Synonyms)

驗證 **IK Analyzer (中文分詞)** 的設定是否符合預期。

| 測試編號 | 測試名稱 | 測試輸入 | 預期結果 (Expected Result) |
| :--- | :--- | :--- | :--- |
| **TC-NLP-001** | **中文分詞準確度** | 搜尋「計算機概論」。 | 系統應能匹配包含「計算機」、「概論」的書籍，而不僅僅是完全符合的字串。 |
| **TC-NLP-002** | **同義詞擴展 (Synonyms)** | 搜尋「DS」或「Algo」。 | 應能搜尋到標題含有「Data Structure」或「Algorithm」的書籍（需在 ES 設定同義詞庫）。 |

---

## 4. 測試資料準備 (Test Data Setup)

執行測試前，需在 Elasticsearch 中索引 (Index) 以下種子資料 (Seed Data)：

```json
[
  {
    "book_id": "b1",
    "title": "計算機概論",
    "course_name": "計算機概論",
    "teacher_name": "王老師",
    "price": 600,
    "status": "AVAILABLE",
    "condition": "NEW"
  },
  {
    "book_id": "b2",
    "title": "Python 程式設計",
    "course_name": "程式設計(一)",
    "teacher_name": "李老師",
    "price": 400,
    "status": "SOLD",  // 用於測試狀態過濾
    "condition": "USED"
  }
]
```



