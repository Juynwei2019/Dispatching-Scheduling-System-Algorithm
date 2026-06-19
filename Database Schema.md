 🍞 麵包工廠智慧派工系統 - 核心資料模型（Database Schema）

本資料模型為階段一的基礎地基，欄位設計已預留未來階段二（甘特圖排程）、階段三（動態權重評分）與階段四（基因演算法）所需的邊界條件（如溫區、最大容忍等待時間）。

## 1. 設備主檔 (`Equipment_Master`)
* 說明：記錄工廠內所有硬體資源（攪拌機、發酵箱、烤爐）的實體物理屬性。

| 欄位名稱 | 資料型態 | 主鍵/外鍵 | 允許空值 | 說明 | 範例 |
| :--- | :--- | :---: | :---: | :--- | :--- |
| `Equipment_ID` | `VARCHAR(50)` | **PK** | ❌ | 設備唯一代碼 | `OVN_01`, `MIX_02` |
| `Equipment_Name` | `NVARCHAR(100)`| — | ❌ | 設備名稱 | `1號雙層烤爐`, `A大攪拌機` |
| `Equipment_Type` | `VARCHAR(20)` | — | ❌ | 設備類型 (`Mixer`/`Fermentation`/`Oven`) | `Oven` |
| `Max_Capacity` | `INT` | — | ❌ | 最大產能/容盤數 | `8` (代表可放 8 盤) |
| `Temp_Range_Min` | `INT` | — | ✔️ | 設備適用最低溫度 (非烤爐則留空) | `150` |
| `Temp_Range_Max` | `INT` | — | ✔️ | 設備適用最高溫度 (非烤爐則留空) | `250` |

---

## 2. 品項工藝主檔 (`Product_Routing_Master`)
* 說明：標準工藝路線（BOM/Routing）。將每種麵包的標準配方製程拆解為多個相依的標準工序。

| 欄位名稱 | 資料型態 | 主鍵/外鍵 | 允許空值 | 說明 | 範例 |
| :--- | :--- | :---: | :---: | :--- | :--- |
| `Routing_ID` | `VARCHAR(50)` | **PK** | ❌ | 工藝路線唯一識別碼 | `R_MILK_TOAST_01` |
| `Product_ID` | `VARCHAR(50)` | — | ❌ | 品項代碼 | `P_MILK_TOAST` |
| `Product_Name` | `NVARCHAR(100)`| — | ❌ | 品項名稱 | `牛奶吐司` |
| `Sequence_No` | `INT` | — | ❌ | 工序順序 (由小到大執行) | `1` (第一道工序) |
| `Process_Type` | `VARCHAR(20)` | — | ❌ | 工序類型 (`Mixer`/`Forming`/`Fermentation`/`Oven`) | `Mixer` |
| `Standard_Duration_Mins`| `INT` | — | ❌ | 標準作業工時（分鐘） | `15` |
| `Target_Temperature` | `INT` | — | ✔️ | 目標烘焙溫度 (僅 Oven 工序需要) | `210` |
| `Max_Wait_Mins_After` | `INT` | — | ❌ | 本工序完成後，**最大容忍等待時間** (防過發關鍵欄位) | `10` (發酵完10分鐘內必須進爐) |

---

## 3. 生產批次表 (`Production_Batch`)
* 說明：記錄每日實際產出的麵糰/產品批次（Batch）。烘焙業以「桶/批」為生產單位而非個。

| 欄位名稱 | 資料型態 | 主鍵/外鍵 | 允許空值 | 說明 | 範例 |
| :--- | :--- | :---: | :---: | :--- | :--- |
| `Batch_ID` | `VARCHAR(50)` | **PK** | ❌ | 批次唯一代碼 | `B20260619_001` |
| `Product_ID` | `VARCHAR(50)` | **FK** | ❌ | 關聯品項代碼 (`Product_Routing_Master`) | `P_MILK_TOAST` |
| `Quantity` | `INT` | — | ❌ | 本批次計畫生產數量 (個/條) | `60` |
| `Due_DateTime` | `DATETIME` | — | ❌ | 物流車發車時間/交期時間 (排程與緊急度計算核心) | `2026-06-19 16:00:00` |
| `Current_Status` | `VARCHAR(20)` | — | ❌ | 批次總體狀態 (`Created`/`Scheduling`/`In_Progress`/`Completed`/`Abnormal`) | `In_Progress` |

---

## 4. 現場製程動態報工表 (`Batch_Process_Log`)
* 說明：MES 現場動態追蹤的核心。記錄每一個生產批次（Batch）當前正在哪一個工作站、哪台設備加工，以及實際進出時間。

| 欄位名稱 | 資料型態 | 主鍵/外鍵 | 允許空值 | 說明 | 範例 |
| :--- | :--- | :---: | :---: | :--- | :--- |
| `Log_ID` | `BIGINT` | **PK** | ❌ | 流水號主鍵 (Identity) | `10234` |
| `Batch_ID` | `VARCHAR(50)` | **FK** | ❌ | 關聯生產批次代碼 (`Production_Batch`) | `B20260619_001` |
| `Sequence_No` | `INT` | — | ❌ | 當前執行的工序順序 (對應 Routing) | `4` (假設代表後發酵) |
| `Equipment_ID` | `VARCHAR(50)` | **FK** | ✔️ | 實際佔用/使用的設備代碼 (等待中可為空) | `OVN_01` |
| `Actual_Start_Time` | `DATETIME` | — | ✔️ | **實際動工/進爐時間** (Track-In 時填入) | `2026-06-19 13:15:22` |
| `Actual_End_Time` | `DATETIME` | — | ✔️ | **實際完工/出爐時間** (Track-Out 時填入) | `2026-06-19 13:27:22` |
| `Status` | `VARCHAR(20)` | — | ❌ | 本道工序狀態 (`Waiting`/`Processing`/`Completed`) | `Processing` |

---

## 🔗 實體關聯邏輯說明 (Entity Relationship Summary)
1. **`Production_Batch`** 透過 `Product_ID` 參照 **`Product_Routing_Master`**，得知該批次要依序跑完哪些 `Sequence_No`。
2. 現場開工時，系統會自動在 **`Batch_Process_Log`** 生成該批次的工序紀錄。
3. 透過動態報工表中的 `Equipment_ID` 關聯 **`Equipment_Master`**，系統便能即時動態分析哪些設備正處於佔用（`Processing`）或閒置狀態。
