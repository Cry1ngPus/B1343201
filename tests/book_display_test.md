# 🧪 書籍展示與交易選項 - 測試計畫 (Book Display Test Plan)

## 1. 測試目標 (Test Objectives)
本文件定義了針對 `Book-Display-Service` 的測試策略與案例，旨在驗證：
* **功能正確性:** 賣家能順利刊登書籍，且買家能看到正確的書籍詳情與交易選項。
* **安全性 (關鍵):** 驗證系統能有效防禦惡意圖片上傳 (Payload Injection)。
* **隱私保護:** 確保 API 回傳的資料中不包含賣家的敏感個人資訊。

---

## 2. 測試環境 (Test Environment)
* **Environment:** Staging (準生產環境)
* **Database:** PostgreSQL (Test Instance with seeded data)
* **Tools:**
    * **Postman / cURL:** 用於 API 請求測試。
    * **OWASP ZAP:** 用於安全性掃描。
    * **Pytest:** 自動化測試腳本。

---

## 3. 測試案例 (Test Cases)

### 3.1 功能性測試 (Functional Testing)

| 測試編號 | 測試名稱 | 前置條件 (Pre-conditions) | 測試步驟 (Test Steps) | 預期結果 (Expected Result) |
| :--- | :--- | :--- | :--- | :--- |
| **TC-BD-001** | **正常書籍刊登流程** | 賣家已登入並持有有效 Token。 | 1. 呼叫 `POST /books`。<br>2. 輸入合法的書名、價格、新舊程度。<br>3. 提供一組有效的圖片 URL。<br>4. 選擇「面交」與「超商取貨」。 | API 回傳 `201 Created`，並返回新書籍的 `book_id` 與狀態 `AVAILABLE`。 |
| **TC-BD-002** | **必填欄位驗證** | 同上。 | 1. 呼叫 `POST /books`。<br>2. **故意漏填** `price` 或 `title`。<br>3. 發送請求。 | API 回傳 `400 Bad Request`，錯誤訊息指出缺少的欄位。 |
| **TC-BD-003** | **價格負數驗證** | 同上。 | 1. 呼叫 `POST /books`。<br>2. 將 `price` 設定為 `-100`。<br>3. 發送請求。 | API 回傳 `400 Bad Request`，提示價格無效。 |
| **TC-BD-004** | **交易選項顯示** | 書籍已成功刊登 (TC-BD-001)。 | 1. 使用買家帳號呼叫 `GET /books/{id}`。<br>2. 檢查 Response 中的 `delivery_options`。 | 回傳的 JSON 包含 `["FACE_TO_FACE", "CONVENIENCE_STORE"]`，且格式正確。 |

### 3.2 安全性測試 (Security Testing - 圖片上傳)

這是針對 **NFR-BD-S-001** (防止網站癱瘓/惡意攻擊) 的關鍵測試。

| 測試編號 | 測試名稱 | 攻擊手法 / 測試輸入 | 預期結果 (Expected Result) |
| :--- | :--- | :--- | :--- |
| **TC-SEC-001** | **非圖片檔案上傳** | 嘗試上傳一個 `.exe` 或 `.sh` 腳本檔案，但偽裝成 `.jpg` 副檔名。 | Image Service 應拒絕簽發 Presigned URL，或在上傳後校驗失敗，書籍刊登請求被拒絕 (`400` 或 `422`)。 |
| **TC-SEC-002** | **圖片 Payload 注入** | 上傳一張合法的 PNG 圖片，但在圖片的 EXIF 或二進制數據尾端注入 **XSS Script** (`<script>alert(1)</script>`)。 | 系統應能偵測並清洗 (Sanitize) 該圖片，或者直接拒絕上傳。 |
| **TC-SEC-003** | **超大檔案攻擊 (DoS)** | 嘗試上傳一個體積超過限制 (例如 > 100MB) 的圖片檔案。 | 系統應在請求 Presigned URL 階段就拒絕，回傳 `413 Payload Too Large`。 |

### 3.3 隱私測試 (Privacy Testing)

這是針對 **NFR-BD-S-002** (買賣雙方隱私保護) 的驗證。

| 測試編號 | 測試名稱 | 測試步驟 | 預期結果 (Expected Result) |
| :--- | :--- | :--- | :--- |
| **TC-PRV-001** | **賣家個資隱藏** | 1. 賣家 (User A) 刊登一本書。<br>2. 買家 (User B) 呼叫 `GET /books/{id}` 獲取該書詳情。<br>3. 檢查回傳的 JSON Payload。 | JSON 中**絕對不可包含** User A 的 `email`, `phone_number` 或真實姓名 (`real_name`)。僅允許顯示 `seller_id` 或 `nickname`。 |

### 3.4 效能測試 (Performance Testing)

針對系統在高負載下的穩定性與響應速度進行驗證。

| 測試編號 | 測試名稱 | 測試場景與條件 | 預期結果 (Expected Result) |
| :--- | :--- | :--- | :--- |
| **TC-PERF-001** | **刊登 API 響應時間** | 1. 模擬單一用戶呼叫 `POST /books`。<br>2. 不包含圖片上傳時間（僅計算 Metadata 寫入）。 | API 平均響應時間應 **< 800ms (毫秒)**。 |
| **TC-PERF-002** | **詳情頁載入速度** | 1. 呼叫 `GET /books/{id}`。<br>2. 該書籍包含 5 張高解析度圖片 URL。 | API 響應時間應 **< 300ms** (圖片由客戶端異步載入，不計入 API 時間)。 |
| **TC-PERF-003** | **高併發壓力測試** | 1. 使用 JMeter 或 k6 模擬 **50 位賣家同時** 執行書籍上架操作。<br>2. 持續時間 5 分鐘。 | 1. 錯誤率 (Error Rate) 為 **0%**。<br>2. 95% 的請求響應時間 (P95) 應低於 **2000ms**。<br>3. 資料庫不應出現 Deadlock。 |

---

## 4. 測試資料準備 (Test Data Setup)

為了執行上述測試，需預先在資料庫中準備以下資料：

```json
// 預設賣家帳號 (用於測試刊登)
{
  "user_id": "seller-001",
  "username": "senior_student_A",
  "role": "SELLER",
  "token": "valid_jwt_token_..."
}

// 預設惡意圖片 (用於安全性測試)
// 檔案: exploit.jpg (內含 PHP webshell 代碼)
```

