# 📚 書籍展示與交易選項 - 技術設計文件 (Book Display SDD)

## 1. Introduction
- **Purpose**: 定義「書籍展示與刊登模組」的技術架構、API 規格、資料模型以及關鍵流程設計。確保將功能需求（如：多圖支援、狀態管理）和非功能性需求（如：圖片安全性、買賣雙方隱私）轉換為可執行的工程方案。
- **Scope**:
    * **In Scope (範圍內)**: 賣家書籍刊登 API (`POST /books`)、買家獲取書籍詳情 API (`GET /books/{id}`)、圖片上傳安全流程、書籍狀態機設計、交易選項動態展示邏輯、隱私中介機制。
    * **Out of Scope (範圍外)**: 實際金流支付與扣款流程（由金流服務模組處理）、站內信箱即時通訊服務。
- **Definitions and Acronyms**:
    * **TDD/SDD**: 技術設計文件 (Technical/Software Design Document)。
    * **Payload**: 惡意輸入或用於測試安全性的資料，這裡特指圖片上傳時防止潛在的惡意程式碼。
    * **Status State Machine**: 書籍從「刊登中」到「已售出」的狀態轉換邏輯。
    * **Presigned URL**: 預簽名 URL，用於安全且有時效性地將檔案直接上傳到雲端儲存空間。
- **References**:
    * [功能需求文件] `../requirement/book_display_requirement.md`
    * [專案總覽] `../README.md`
---
## 2. System Overview

### System Description (系統描述)
書籍展示與刊登模組是一個使用者導向的子系統，負責處理賣家（學長）的產品資訊輸入、圖片處理、狀態管理，以及向買家（學弟）呈現書籍詳情和可用的交易選項。它是連接賣家與訂單處理系統的門戶。

---

### Design Goals (設計目標)

* **Usability (可用性):** 實現簡潔直觀的刊登流程，減少賣家上架時間 (NFR-BD-U-001)。
* **Security (安全性):** 嚴格執行圖片上傳檢查（防止惡意 payload），並確保買賣雙方在交易過程中全程保持隱私 (NFR-BD-S-001, NFR-BD-S-002)。
* **Maintainability (可維護性):** 採用清晰定義的狀態機 (State Machine) 來管理書籍狀態，以利於日後追蹤和維護交易流程。
* **Performance (效能):** 優化圖片處理和資料載入，確保書籍展示頁面快速響應 (NFR-BD-P-001)。

---

### Architecture Summary (架構摘要)

本模組預計採用**微服務架構 (Microservices)** 的其中一個服務來實現。

* **服務名稱:** `Book-Listing-Service`
* **職責:** 專注於書籍資料的 CRUD (創建、讀取、更新、刪除) 操作、庫存狀態管理，以及處理與**專門圖片服務 (Image Service)** 的互動。
* **依賴:** 依賴獨立的 **Image Service** 處理圖片的格式檢查和儲存；依賴 **Notification Service** 在購買時通知賣家 (FR-BD-007)。

---

### System Context Diagram (上下文交換圖)

```mermaid
graph LR
    %% 核心系統 (視為一個單一黑箱)
    System("Book Listing Service<br>(書籍刊登系統)")

    %% 外部實體 (External Entities)
    Seller("賣家 - 學長")
    Buyer("買家 - 學弟")
    ImageStorage("圖片儲存服務")
    NotificationService("通知服務")
    OrderService("訂單服務")

    %% 賣家操作
    Seller -->|刊登/更新書籍資料| System
    System -->|請求上傳 URL| ImageStorage
    ImageStorage -->|傳回 URL| System
    Seller -. 上傳圖片 .-> ImageStorage

    %% 買家瀏覽
    Buyer -->|獲取書籍詳情| System

    %% 交易與狀態
    Buyer -->|建立訂單| OrderService
    OrderService -->|更新書籍狀態| System
    System -->|賣家通知| NotificationService
    
    %% 隱私
    System -. 隱藏個資 .- Buyer
```

---
