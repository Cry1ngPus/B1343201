# 🔍 書籍搜尋系統 - 技術設計文件 (Book Search SDD)
---
## 1. Introduction
---
- **Purpose**: 定義「書籍搜尋模組」的技術架構、索引策略與查詢邏輯。旨在解決傳統關聯式資料庫無法滿足的高效能全文檢索與多維度篩選（書名、課程、老師）需求。
- **Scope**:
    * **In Scope (範圍內)**: 書籍搜尋 API (`GET /search/books`)、搜尋引擎 (Elasticsearch) 的索引結構設計 (Mapping)、加權排序演算法 (Ranking Logic)、資料庫與搜尋引擎之間的資料同步機制 (Data Sync)。
    * **Out of Scope (範圍外)**: 書籍的實際交易處理、圖片儲存細節（由 `book_display` 模組處理）。
- **Definitions and Acronyms**:
    * **Elasticsearch (ES)**: 分散式搜尋與分析引擎，本系統的核心組件，用於全文檢索。
    * **Inverted Index (倒排索引)**: 搜尋引擎使用的核心資料結構，用於快速將關鍵字映射到文檔。
    * **Relevance Score (關聯性分數)**: 根據匹配程度（如 TF-IDF/BM25 演算法）計算出的分數，用於搜尋結果排序。
    * **CDC (Change Data Capture)**: 變更資料擷取，指即時捕捉資料庫變更並同步到搜尋引擎的技術模式。
- **References**:
    * [功能需求文件] `../requirement/book_search_requirement.md`
    * [書籍展示設計] `../design/book_display_design.md` (定義了書籍原始資料來源)
    * [專案總覽] `../README.md`

---
## 2. System Overview
---
### System Description (系統描述)
書籍搜尋系統是一個專門優化讀取效能 (Read-Optimized) 的服務。它從主資料庫中同步書籍資料，建立全文索引，並提供一個低延遲的查詢介面，讓買家能透過書名、課程名稱、老師姓名等模糊關鍵字快速找到所需書籍。

### Design Goals (設計目標)

* **Performance (效能):** 確保 95% 的搜尋請求能在 **500ms** 內返回結果 (NFR-BS-P-001)。
* **Relevance (精準度):** 實作加權排序，確保「書名完全匹配」的結果排在「課程匹配」之前 (FR-BS-004)。
* **Freshness (即時性):** 書籍狀態變更（如已售出）必須在數秒內反映在搜尋結果中，避免買家看到無效庫存 (NFR-BS-A-001)。
* **Scalability (擴展性):** 架構需支援水平擴展，以應對未來書籍數量增長。

### Architecture Summary (架構摘要)

本模組採用 **CQRS (命令查詢職責分離)** 模式的變體：

* **寫入端 (Write Side):** `Book-Listing-Service` 負責寫入 Primary DB (PostgreSQL)。
* **同步端 (Sync Side):** 透過 **Sync Worker** 或事件驅動機制，將變更異步推送到 Elasticsearch。
* **讀取端 (Read Side):** `Book-Search-Service` 直接查詢 Elasticsearch 來獲取搜尋結果。

### System Context Diagram (上下文交換圖)

```mermaid
graph LR
    %% 核心搜尋系統
    System("Book Search Service<br>(書籍搜尋系統)")

    %% 外部實體
    Buyer("買家 - 學弟")
    ListingService("書籍刊登服務")
    PrimaryDB("主資料庫 (PostgreSQL)")
    Elasticsearch("搜尋引擎 (Elasticsearch)")

    %% 讀取流程 (搜尋)
    Buyer -->|1. 輸入關鍵字/篩選| System
    System -->|2. 查詢 Query DSL| Elasticsearch
    Elasticsearch -->|3. 返回排序結果| System
    System -->|4. 返回 JSON 列表| Buyer

    %% 同步流程 (Indexing)
    ListingService -.->|寫入書籍數據| PrimaryDB
    PrimaryDB -.->|資料變更 CDC| System
    System -.->|更新索引| Elasticsearch
```
---
## 3. Architectural Design
---
### System Architecture Diagram (系統架構圖)

```mermaid
graph TD
    %% 定義節點
    User[Buyer - 買家]
    Gateway[API Gateway]
    SearchService[Book Search Service]
    ES[(Elasticsearch Cluster)]
    
    subgraph Data Synchronization
        PrimaryDB[(Primary DB - PostgreSQL)]
        SyncWorker[Sync Worker / Logstash]
    end
    
    ListingService[Book Listing Service]

    %% 搜尋流程 (Read Path)
    User -->|GET /search| Gateway
    Gateway -->|Route| SearchService
    SearchService -->|Query DSL| ES
    ES -->|Ranked Results| SearchService

    %% 索引流程 (Write/Sync Path)
    ListingService -->|Insert/Update| PrimaryDB
    PrimaryDB -.->|Change Events| SyncWorker
    SyncWorker -->|Index Document| ES
```
### Component Breakdown (組件拆解)

| 組件名稱 (Component) | 職責 (Responsibilities) | 互動對象 (Interactions) |
| :--- | :--- | :--- |
| **Search API Handler** | 接收前端的搜尋請求（關鍵字、篩選條件、分頁）；執行輸入驗證與預處理（例如：去除特殊字符、關鍵字正規化）。 | 接收來自 **API Gateway** 的流量；將請求轉發給 **Query Builder**。 |
| **Query Builder** | 將使用者的搜尋意圖（如：「演算法 老師A」）轉換為 **Elasticsearch Query DSL**。負責設定權重（Boosting），例如讓 `title` 欄位的權重高於 `description`。 | 被 **Search API Handler** 呼叫；向 **Elasticsearch** 發送查詢請求。 |
| **Data Sync Worker** | 負責維持 Primary DB 與 Elasticsearch 之間的資料一致性。監聽資料庫變更或定期輪詢，將最新的書籍狀態（如：已售出）更新到索引中。 | 讀取 **Primary DB** 的變更日誌；對 **Elasticsearch** 執行 Index/Update 操作。 |

### Technology Stack (技術棧)

| 類別 (Category) | 技術選型 (Technology) | 選擇理由 (Rationale) |
| :--- | :--- | :--- |
| **Search Engine** | **Elasticsearch** (v8+) | 業界標準的全文檢索引擎，支援倒排索引、模糊搜尋 (Fuzzy Search) 與高效能的聚合分析，滿足 500ms 內回傳的需求。 |
| **Backend Framework** | **Python (FastAPI)** 或 **Node.js** | 輕量且高效，適合處理搜尋請求的 JSON 轉換與邏輯處理。 |
| **Sync Mechanism** | **Logstash** 或 **Custom Worker** | 用於實現資料庫到搜尋引擎的 ETL (Extract, Transform, Load) 流程，確保資料新鮮度 (Freshness)。 |
| **Analyzer** | **IK Analyzer** (中文分詞器) | 針對中文書籍名稱（如「計算機概論」）進行正確斷詞，避免搜尋不到相關書籍的問題。 |

### Data Flow and Control Flow

```mermaid
sequenceDiagram
    autonumber
    participant Buyer as Buyer (買家)
    participant API as Search API
    participant Builder as Query Builder
    participant ES as Elasticsearch

    Note over Buyer, API: 讀取路徑 (Read Path)
    Buyer->>API: 1. 發送搜尋請求 (GET /search?q=演算法)
    
    API->>API: 2. 輸入驗證與清洗 (Input Sanitization)
    
    API->>Builder: 3. 請求構建查詢 (Build Query)
    Note right of Builder: 應用權重策略 (Boosting)<br/>Title^3, Course^1.5
    Builder-->>API: 4. 返回 Elasticsearch Query DSL

    API->>ES: 5. 執行搜尋 (Execute Search)
    ES-->>API: 6. 返回排序後的結果 (Ranked Hits)

    API-->>Buyer: 7. 返回書籍列表 JSON
```

