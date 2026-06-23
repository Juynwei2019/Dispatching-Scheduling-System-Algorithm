# 🍞 麵包工廠智慧派工系統 - 系統需求規格書 (System Requirements Specification, SRS)

## 📝 1. 專案概述 (Project Overview)
本專案旨在為麵包工廠開發一套**智慧派工與動態調度系統**。烘焙製造業面臨物料高度時效性（麵糰過發報廢）與高換線能耗成本（烤爐降溫極慢）的特殊物理限制。本系統透過雙層自適應演算法架構，實現無需人工維護權重的秒級即時調度，以達到零交期延遲、零物料報廢與極小化能耗之目標。

---

## 🎯 2. 漸進式開發階段規劃 (Incremental Development Phases)

### 📌 階段一：基礎工程與資料實體化 (打好地基)
* **核心目標：** 將工廠實體世界（設備、配方工藝）數位化。
* **功能範疇：** 建立設備主檔、品項工藝主檔、生產批次表，以及簡易 MES 現場報工介面（提供進站/出站按鈕）。
* **資料效益：** 開始收集現場實際加工時間，利用大數據滾動修正標準工時（UPH）。

### 📌 階段二：啟發式規則與看板排程 (靜態排程)
* **核心目標：** 實現自動化日排程大綱，取代人工手寫排班。
* **功能範疇：** 導入兩階段排程法（訂單同品項/同交期窗口自動群組化，再依據 EDD+SPT 混合規則進行時間軸逆向推算）。
* **交付物：** 生管排程動態甘特圖、現場工作站依時間排序的靜態任務清單（Dispatch List）。

### 📌 階段三：事件驅動與動態權重評分 (應變能力)
* **核心目標：** 讓系統具備秒級現場見招拆招的動態應變能力。
* **功能範疇：** 監聽現場事件（報工完工、機台故障、特急單插單）。當設備閒置時，自動調用「自適應動態評分公式」即時計分，最高分者優先指派。
* **安全機制：** 實作攪拌站全線連動斷路機制（Circuit Breaker），後端塞車時前端自動限制投料。

### 📌 階段四：全域優化與基因演算法 (智慧大腦)
* **核心目標：** 引入 AI 數學規劃尋找全天宏觀層級的最優解。
* **功能範疇：** 開班前運行基因演算法（GA）產出今日全廠能耗最低、交期最完美的 Baseline 骨架，並與階段三的動態調度引擎完美對接（宏觀有計畫，微觀有彈性）。

---

## 🗄️ 3. 核心資料模型設計 (Core Data Model)

### 3.1 設備主檔 (`Equipment_Master`)
| 欄位名稱 | 資料型態 | 主鍵/外鍵 | 允許空值 | 說明 |
| :--- | :--- | :---: | :---: | :--- |
| `Equipment_ID` | `VARCHAR(50)` | **PK** | ❌ | 設備唯一代碼 (如 `OVN_01`, `MIX_02`) |
| `Equipment_Name` | `NVARCHAR(100)`| — | ❌ | 設備名稱 (如 1號雙層烤爐) |
| `Equipment_Type` | `VARCHAR(20)` | — | ❌ | 設備類型 (`Mixer`/`Fermentation`/`Oven`) |
| `Max_Capacity` | `INT` | — | ❌ | 最大產能/容盤數 (如可放 8 盤) |
| `Temp_Range_Min` | `INT` | — | ✔️ | 設備適用最低溫度 (非烤爐則留空) |
| `Temp_Range_Max` | `INT` | — | ✔️ | 設備適用最高溫度 (非烤爐則留空) |
| `Current_State` | `VARCHAR(20)` | — | ❌ | 設備即時狀態 (`Idle`/`Processing`/`Down`) |
| `Last_Processed_Temp`| `INT` | — | ✔️ | 記錄該烤爐最後一次運作的實際溫度 |

### 3.2 品項工藝主檔 (`Product_Routing_Master`)
| 欄位名稱 | 資料型態 | 主鍵/外鍵 | 允許空值 | 說明 |
| :--- | :--- | :---: | :---: | :--- |
| `Routing_ID` | `VARCHAR(50)` | **PK** | ❌ | 工藝路線唯一識別碼 |
| `Product_ID` | `VARCHAR(50)` | — | ❌ | 品項代碼 (如 `P_MILK_TOAST`) |
| `Product_Name` | `NVARCHAR(100)`| — | ❌ | 品項名稱 (如 牛奶吐司) |
| `Sequence_No` | `INT` | — | ❌ | 工序順序 (由小到大執行，如 1:攪拌, 2:後發...) |
| `Process_Type` | `VARCHAR(20)` | — | ❌ | 工序類型 (`Mixer`/`Forming`/`Fermentation`/`Oven`) |
| `Standard_Duration_Mins`| `INT` | — | ❌ | 標準作業工時 (分鐘) |
| `Target_Temperature` | `INT` | — | ✔️ | 目標烘焙溫度 (僅 Oven 工序需要) |
| `Max_Wait_Mins_After` | `INT` | — | ❌ | 本工序完成後，最大容忍等待時間 (防過發靈魂欄位) |

### 3.3 生產批次表 (`Production_Batch`)
| 欄位名稱 | 資料型態 | 主鍵/外鍵 | 允許空值 | 說明 |
| :--- | :--- | :---: | :---: | :--- |
| `Batch_ID` | `VARCHAR(50)` | **PK** | ❌ | 批次唯一代碼 |
| `Product_ID` | `VARCHAR(50)` | **FK** | ❌ | 關聯品項代碼 |
| `Quantity` | `INT` | — | ❌ | 本批次計畫生產數量 (個/條) |
| `Due_DateTime` | `DATETIME` | — | ❌ | **物流車發車時間/交期時間** (排程計分核心) |
| `Batch_Group_ID` | `VARCHAR(50)` | — | ✔️ | 階段二系統自動群組化後的群組 ID |
| `Current_Status` | `VARCHAR(20)` | — | ❌ | 批次總體狀態 (`Created`/`Scheduling`/`In_Progress`/`Completed`/`Abnormal`) |

### 3.4 現場製程動態報工表 (`Batch_Process_Log`)
| 欄位名稱 | 資料型態 | 主鍵/外鍵 | 允許空值 | 說明 |
| :--- | :--- | :---: | :---: | :--- |
| `Log_ID` | `BIGINT` | **PK** | ❌ | 流水號主鍵 (Identity) |
| `Batch_ID` | `VARCHAR(50)` | **FK** | ❌ | 關聯生產批次代碼 |
| `Sequence_No` | `INT` | — | ❌ | 當前執行的工序順序 (對應 Routing) |
| `Equipment_ID` | `VARCHAR(50)` | **FK** | ✔️ | 實際佔用/使用的設備代碼 |
| `Actual_Start_Time` | `DATETIME` | — | ✔️ | **實際動工/進爐時間** (Track-In 時填入) |
| `Actual_End_Time` | `DATETIME` | — | ✔️ | **實際完工/出爐時間** (Track-Out 時填入) |
| `Enter_Waiting_Time` | `DATETIME` | — | ✔️ | 記錄該工序何時轉為 Waiting 狀態 (計算乾等時間) |
| `Dynamic_Score` | `DECIMAL(5,2)` | — | ✔️ | 階段三演算法即時填入的當前動態得分 |
| `Status` | `VARCHAR(20)` | — | ❌ | 本道工序狀態 (`Waiting`/`Processing`/`Completed`) |

---

## 📐 4. 自適應動態調度演算法規格 (Adaptive Algorithm Specification)

本系統採去中心化維護思維，權重配比較由環境壓力動態驅動。

### 4.1 第一層：宏觀權重自適應引擎 (Softmax 壓力歸一化)
系統每 30 秒自動掃描全廠 $N$ 筆在製品，動態調整交期、能耗與品質權重，且滿足 $w_1 + w_2 + w_3 = 1.0$。

1.  **交期壓力因子 ($P_{	ext{cr}}$) —— Sigmoid S型曲線：**
    * 首先計算全廠平均臨界比率 $\overline{	ext{CR}}$ = $rac{1}{N} \sum_{i=1}^{N} (rac{	ext{距離發車剩餘時間}}{	ext{剩餘總標準加工工時}})$。
    * 帶入公式：$w_1 = w_{	ext{min}} + rac{w_{	ext{max}} - w_{	ext{min}}}{1 + e^{k \cdot (\overline{	ext{CR}} - 	ext{Threshold})}}$ （全廠大遲到時，交期權重 $w_1$ 自動封頂至 0.7）。
2.  **能耗壓力因子 ($P_{	ext{setup}}$) —— 連動台電電費：**
    * $P_{	ext{setup}} = P_{	ext{Electric}} 	imes (1.0 + P_{	ext{Idle}})$
    * 當處於夏季下午尖峰電費時段 ($P_{	ext{Electric}} = 1.0$) 且全廠時間充裕產線空閒時，能耗壓力飆高，主動推高能耗權重 $w_2$ 啟動節能模式。
3.  **品質壓力因子 ($P_{	ext{ferment}}$) —— 現場在製品飽和度：**
    * 與當前發酵箱外排隊等烤爐的 Batch 數量成正比，防止集體報廢。

### 4.2 第二層：微觀即時評分公式 (Micro-Dispatching Score)
當特定烤爐 $M$ 空閒時，針對排隊中（`Waiting`）的批次 $J$ 計算綜合得分，最高分者優先派工進爐：

$$	ext{Score}(J, M) = w_1 \cdot S_{	ext{CR}}(J) + w_2 \cdot S_{	ext{Setup}}(J, M) + w_3 \cdot S_{	ext{Ferment}}(J)$$

* **個體交期得分 $S_{	ext{CR}}(J)$：** $	ext{CR}_J \le 0.5$ 直接給滿分 1.0；$	ext{CR}_J > 1.5$ 給 0 分；其餘區間線性歸一化。
* **能耗得分自適應 $S_{	ext{Setup}}(J, M)$：** 採用**連續指數衰減函數 $e^{-\lambda \cdot \Delta T}$**。
    * 目標溫度 $\ge$ 爐溫（升溫）：$\lambda = 0.01$（衰減慢，得分高）。
    * 目標溫度 $<$ 爐溫（降溫）：$\lambda = 0.05$（降溫散熱極慢，能耗代價高，得分迅速探底）。
* **防過發品質得分 $S_{	ext{Ferment}}(J)$ —— 核心防呆：**
    * $S_{	ext{Ferment}}(J) = \min\left(1.0, \left( rac{	ext{當前已等待時間}}{	ext{最大容忍等待時間}} 
ight)^3
ight)$
    * **立方級別暴增：** 當麵糰逼近過發報廢邊緣（如已等 18 分鐘/上限 20 分鐘），該項得分呈爆發性飆升至接近 1.0，強制壓過能耗與交期劣勢，躍升看板首位以搶救物料。
