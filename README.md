# 國立宜蘭大學二手書交易平台 (NIU Second-Hand Book Exchange Platform)

## 📖 專案總覽 (Overview)

本專案旨在為宜蘭大學的師生建立一個高效、簡單操作且重視隱私的二手教科書與課堂筆記交易平台。
包含完整的軟體需求文件(SRS,Software Requirements Specification),軟體設計文件(SDD,Software Design Document)及測試文件(Test Document)

## 試圖解決之需求
* **賣家（學長）需求：** 解決修畢者無法順利出售書籍，或難以在 DCARD 等平台管理上架和回覆買家訊息的問題，提供簡單管理書籍上架的單一平台。
* **買家（學弟）需求：** 確保買家能在短時間內找到目標書籍，並在交易過程中得到隱私保護。
---
## 關鍵技術決策 (Planned Tech Decisions)

| 領域 | 潛在技術棧 | 設計考量 |
| :--- | :--- | :--- |
| **搜尋系統** | Elasticsearch / Full-Text Search | 必須確保能在可忍受時間內呈現書籍結果，滿足快速搜尋需求。 |
| **圖片儲存** | Cloud Storage (e.g., AWS S3) | 用於安全存儲 PNG, JPG 格式的產品照片，並確保輸入欄位無安全隱患。 |
| **後端服務** | Node.js / Python (待定) | 須能處理賣家帳號連動通知、隱私交易機制等複雜邏輯。 |
---
## 1. 🔍 書籍搜尋系統 (Book Search System)

* **文件位置：** `design/book_search_design.md`
* **重點關注：** 實現透過書名、對應課程、授課老師等多維度的**高效能搜尋**，並確保搜尋導覽正常運作，包含測試與安全檢查。
* **預計時程：** 預計功能與預覽功能基礎架構需耗費 4 小時；測試與安全檢查需耗費 3 小時。

## 2. 🖼️ 書籍展示與交易選項 (Book Display and Transaction Options)

* **文件位置：** `design/book_display_design.md`
* **重點關注：** 確保產品照片、書籍細節、新舊程度等資訊正常上傳與展示。同時設計買家隱私保護介面，並規劃**面交以外的多種取貨選項**。
* **預計時程：** 預計上架功能與展示內容的基礎架構需耗費 3 小時；測試與安全檢查需耗費 3 小時。

---

## 📁 專案結構導覽 (Project Structure Guide)
design/: 存放軟體設計文件 (SDD)。

requirement/: 存放基於學長姐弟需求拆解的詳細需求文件(SRS)。

tests/: 存放每個功能規劃的測試案例與安全檢查清單(Test Document)。

README.md: 專案總覽文件 (即此文件)。

## 🤝 貢獻與聯絡 (Contribution and Contact)

歡迎任何對於本專案的建議或貢獻。

* **貢獻指南：** 請先 Fork 並基於`main`分支建立新Branch。
* **聯絡方式：** [NIU CSIE B1343201]
