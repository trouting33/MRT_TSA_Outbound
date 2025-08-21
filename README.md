# Gogoro Data Office — Data Challenge README

> Project: Taipei Metro Passenger Hourly Statistics (2017-01 ~ 2025-06)

---

## 1) 專案總覽 (Executive Summary)
本專案以「臺北捷運各站分時進出量統計」資料為核心，完成：
- **資料前處理 (Data preprocessing)**：將多檔原始 CSV 清理、統一欄位型態、合併為可分析之長表格式。
- **特徵工程 (Feature engineering)**：建立時間/節日等日曆特徵，供後續分析與建模使用。
- **模型與驗證 (Model & Validation)**：以 **Synthetic Control Method (SCM)** 進行事件影響評估，並引入 **Herfindahl–Hirschman Index (HHI)** 作為捐贈站(donor pool)權重集中度的正則化指標，降低單一 donor 過度主導的風險；同時採用交叉驗證與安慰劑 (placebo) 分析。

核心產出包含：
- `data/processed/exits_by_station_hour.csv`：各站逐時出站人次彙整表（後續分析/模型的主資料）。
- `notebooks/data_wrangling.ipynb`：清理與彙整流程。
- `notebooks/SCM_HHI_2.ipynb`：SCM + HHI 的影響評估流程與圖表。
- `slides/`：簡報（Executive Summary、Objective、Methodology、Key Results、Conclusion）。

---

## 2) 專案結構 (Repository Structure)
```
.
├─ data/
│  ├─ 臺北捷運每日分時各站OD流量統計資料_20xx_xx.csv                        # 原始資料（臺北捷運各站分時進出量統計）
│  ├─ 出國旅客按目的地統計.csv                                             # 外部資料
│  ├─ exits_by_station_hour.csv                                          # 主訓練/分析資料
│  └─ entries_by_station_hour.csv                                        # optional
├─ notebooks/
│  ├─ data_wrangling.ipynb             # 資料清理與彙整
│  └─ SCM_HHI.ipynb                    # SCM + HHI 分析與驗證
├─ slides/                             # 簡報（PDF 或 PPTX）
├─ requirements.txt                    # 相依套件清單
└─ README.md                           # 本文件
```

---

## 3) 環境需求 (Environment)

```
- Python ≥ 3.10
- 建議以 `venv` 或 `conda` 建立隔離環境

範例安裝：
```bash
# 建立虛擬環境（venv 範例）
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

# 安裝相依
pip install -r requirements.txt
```



---

## 4) 資料來源與欄位 (Data Source & Schema)
- **資料來源**：臺北捷運各站分時進出量統計（2017-01 ~ 2025-06）
- **原始欄位**：`[日期, 時段, 進站, 出站, 人次]`
  - `日期`：YYYY-MM-DD
  - `時段`：0–23 代表小時
  - `進站/出站`：站名
  - `人次`：整數（該站該時段之進/出站人次）

### 主輸出表：`exits_by_station_hour.csv`
- 粒度：**站別 × 小時**
- 主要欄位：
  - `timestamp`：UTC+8 時區之小時時間戳 (e.g., `2019-05-01 08:00:00`)
  - `station`：站名（與原始一致或標準化後名稱）
  - `exits`：該站該小時之出站人次（`人次` 中的「出站」彙整）
  - （可選）日曆特徵：`dow`, `is_weekend`, `is_holiday`, `month`, `hour` 等

---

## 5) 前處理流程 (notebooks/data_wrangling.ipynb)
此 Notebook 的目標是將原始多檔資料整併為乾淨、統一的長表：
1. **讀取與型別統一**：將 `日期` 轉為 `datetime`，`時段` 轉為整數，再合併為小時級 `timestamp`。
2. **資料清理**：過濾缺漏與不合理值、修補常見站名異拼（若有）、移除重複列。
3. **長表化**：聚焦「出站」人次，彙整為 `station` × `timestamp` 的長表。
4. **特徵工程（可選）**：
   - 時間特徵：`dow(weekday)`, `hour`, `month`, `is_weekend`。
   - 節假日特徵：以臺灣國定假日產生 `is_holiday`（或以自行維護表匯入）。
5. **輸出**：將結果寫出至 `data/processed/exits_by_station_hour.csv`，供 SCM 與其他分析使用。

> 若您有 102 份（或更多）CSV，建議以 DuckDB/Polars 批次匯入與彙總，能大幅縮短執行時間。

---

## 6) 模型與驗證 (notebooks/SCM_HHI_2.ipynb)
此 Notebook 以 **合成控制法 (SCM)** 量化某事件/政策/方案對特定「測試站 (treated station)」之出站量影響。

### 6.1 方法概述
- **SCM**：以一組「捐贈站 (donor pool)」的加權組合，擬合測試站在**干預前**的趨勢，作為**反事實 (counterfactual)**，再於**干預後**比較實際 vs. 反事實以估計 **lift/影響**。
- **HHI 正則化**：計算 donor 權重向量 \( w \) 的 **Herfindahl–Hirschman Index (HHI) = \(\sum_i w_i^2\)**，以 \(\text{score} = \text{RMSE}_{\text{pre}} + \lambda\, \text{HHI}\) 作為選模指標，降低單一 donor 過度主導。
- **模型選擇與驗證**：
  - 在訓練窗 (pre) 以交叉驗證/滾動窗口調參（包含 \(\lambda\)、donor 數量/名單等）。
  - **Placebo 測試**（in-space / in-time）：用其他未處理站/時段模擬「假干預」，檢驗 lift 是否顯著高於隨機波動。

### 6.2 Notebook 主要步驟
1. **載入資料**：讀取 `exits_by_station_hour.csv`，過濾目標站與 donor pool。
2. **視窗設定**：明確指定 `pre_period = [t0, t1)` 與 `post_period = [t1, t2]`。
3. **權重估計**：
   - 約束：權重非負、且總和為 1（傳統 SCM），或允許鬆綁但加以正則化（延伸做法）。
   - 以 \(\text{RMSE}_{\text{pre}} + \lambda\, \text{HHI}\) 選模（可網格搜尋 \(\lambda\)）。
4. **影響評估**：
   - 生成反事實序列，計算 **逐時 lift** 與 **事件期累積 lift**。


> **解讀指引**：若測試站在 post 期間與反事實有穩健且顯著的正差異（且 placebo 顯示罕見），即可支持「干預造成出站量提升」的結論；反之亦然。

---

## 7) 執行指南 (How to Run)
1. **準備資料**：將原始 CSV 放入 `data/`。
2. **跑前處理**：開啟並執行 `notebooks/data_wrangling.ipynb` → 產生 `data/exits_by_station_hour.csv`。
3. **設定事件窗與站點**：於 `notebooks/SCM_HHI.ipynb` 之「參數設定」區塊，指定 `treated_station`、`pre_period`、`post_period`、`lambda_grid`、`donor_pool` 規則等。
4. **執行 SCM**：依 Notebook 指示逐步執行全部 cells，完成模型選擇、影響估計、圖表輸出。
5. **檢視輸出**：查看 `figures/` 與 Notebook 結尾的結果小結。


---

## 8) 再現性 (Reproducibility)
- 指定 Python/套件版本於 `requirements.txt`。
- 在 Notebook 開頭固定隨機種子（e.g., `np.random.seed(42)`）。
- 明確紀錄 pre/post 期間與站點清單；所有圖表以程式自動產生，避免手動調整。

---

## 9) 注意事項與假設 (Assumptions)
- 站名標準化：若官方資料曾更名/異拼，需以對照表統一。
- 事件獨立性：SCM 假設 post 期間主要變化來自目標干預；若同窗出現其他重大事件，需額外控制或在討論中聲明限制。
- 溢出效應 (spillover)：若干預造成鄰近站/路線影響，donor 選取應避開高度相關之站點，或於敏感度分析中測試不同 donor 組合。


---

## 10) 聯絡 (Contact)
如有任何問題，歡迎與我聯繫：
- Chien-Ping (Paul) Wu — `paulwu.apply@gmail.com`

