# TODO

Priority legend / 優先級說明:
- `P1` = current critical path / 目前主線關鍵工作
- `P2` = important follow-up / 重要後續工作
- `P3` = later improvement / 後續優化

## Current Direction / 目前主方向

根據目前 4 份 TODO 整理後，專案的優先順序應調整為：

1. 先修正 `financialdatasets.ai` 現行 API 不相容問題，解除 `prices` 與 `news` 的 `HTTP 400` 風險
2. 在不改動 agent 呼叫方式的前提下，抽象化 financial data provider
3. 先導入 `FMP (Financial Modeling Prep)` 作為主替代方案
4. 補上本地快取、測試與可觀測性
5. 最後再做 expert system 權重與回測優化

目前不建議把「研究更多資料源」、「自建資料管線」、「專家系統擴充」放在最前面，因為它們都依賴前兩步先穩定下來。

## Phase 1 / 修正現有 Financial Datasets 整合

### Prices API

- [ ] `P1` 更新 `src/tools/api.py` 的 `get_prices()` request 組裝邏輯
- [ ] `P1` 將 endpoint 統一為 `https://api.financialdatasets.ai/prices`
- [ ] `P1` 只送官方文件允許的參數：`ticker`、`interval`、`start_date`、`end_date`
- [ ] `P1` 停止送出 `interval_multiplier`
- [ ] `P1` 保留既有函式簽名與 `interval=day` 行為，避免上層 agent/backtest 改寫
- [ ] `P2` 視需要補 log，讓非 200 回應可追查實際 request 問題
- [ ] `P1` 驗證 `PriceResponse` 與價格轉 DataFrame 流程仍相容

### News API

- [ ] `P1` 更新 `src/tools/api.py` 的 `get_company_news()` request 組裝邏輯
- [ ] `P1` 將 endpoint 統一為 `https://api.financialdatasets.ai/news`
- [ ] `P1` 只送官方文件允許的參數：`ticker`、`limit`
- [ ] `P1` 將 `limit > 10` 自動 clamp 到 `10`
- [ ] `P1` 停止送出 `start_date`、`end_date`
- [ ] `P1` 保留既有函式簽名，避免上層呼叫全面改寫
- [ ] `P1` 對被忽略的歷史篩選與過大 `limit` 加上 warning log
- [ ] `P1` 視需要調整 cache key，避免被忽略參數造成誤導性快取

### Phase 1 Validation

- [ ] `P1` 為 `get_prices()` 補 request 組裝測試，確認不再送出 `interval_multiplier`
- [ ] `P1` 為 `get_company_news()` 補 request 組裝測試，確認不再送出 `start_date`、`end_date`
- [ ] `P1` 驗證 `limit` 會被 clamp 到 `10`
- [ ] `P1` 驗證 rate limit retry 行為不受影響
- [ ] `P1` 驗證至少一條經過 `get_prices()` 的 agent 或 backtest 路徑可正常完成
- [ ] `P1` 驗證 news/sentiment 相關 agent 在最多只有 `10` 筆新聞時不會報錯

## Phase 2 / 抽象化 Financial Data Provider

- [ ] `P1` 在 `src/tools/api.py` 前加一層 provider dispatch
- [ ] `P1` 保持 agents 對既有函式的呼叫方式不變
- [ ] `P1` 新增 `FINANCIAL_DATA_PROVIDER` 環境變數
- [ ] `P1` 預設 provider 設為 `fmp`
- [ ] `P2` 明確定義 provider adapter 輸出契約，維持下列 model shape 不變：
  - `CompanyNews`
  - `FinancialMetrics`
  - `Price`
  - `InsiderTrade`
  - `LineItem`
- [ ] `P2` 補 provider 層測試，確認切換 provider 時不影響上層 agent

## Phase 3 / 導入 FMP 作為主替代方案

### Why FMP / 為什麼先做 FMP

目前結論維持：
- `FMP` 是最接近單一供應商替代，成本、覆蓋率、遷移難度平衡最好
- `Alpha Vantage` 適合低預算 demo，但不適合這個 repo 當主供應商
- `EODHD` 可作為全球市場導向備選
- `Polygon` 暫不列為本次低成本優先方案

### FMP Implementation

- [ ] `P1` 先完成 `get_prices()` 的 FMP adapter
- [ ] `P1` 先完成 `get_company_news()` 的 FMP adapter
- [ ] `P1` 先完成 `get_financial_metrics()` 的 FMP adapter
- [ ] `P1` 先完成 `get_insider_trades()` 的 FMP adapter
- [ ] `P1` 先完成 `get_market_cap()` 的 FMP adapter
- [ ] `P2` adapter 內處理欄位名稱差異，對上層輸出 shape 保持不變
- [ ] `P2` 盡量沿用現有 cache 行為

### Line Items

- [ ] `P1` 保留 `search_line_items()` 對上層的函式介面
- [ ] `P1` 底層改用 FMP 財報資料映射成 `LineItem`
- [ ] `P2` 不追求與 `financialdatasets.ai` 100% 對等，只覆蓋目前 agents 會用到的關鍵欄位

### Config and Docs

- [ ] `P2` 新增 `FMP_API_KEY`
- [ ] `P2` 前端 API key 設定支援 `FMP_API_KEY`
- [ ] `P2` 後端讀 key 邏輯支援新 provider
- [ ] `P2` 文件改為說明 provider 切換方式
- [ ] `P3` 保留未來加入 `ALPHA_VANTAGE_API_KEY` / `EODHD_API_KEY` 的擴充空間

### Phase 3 Validation

- [ ] `P1` 驗證 `get_company_news()` 能映射出正確的 `CompanyNews`
- [ ] `P1` 驗證 `news_sentiment_agent()` 在缺少 sentiment 時仍能走既有 LLM fallback
- [ ] `P1` 驗證 `get_prices()` 的 OHLCV 與時間排序符合目前 DataFrame 假設
- [ ] `P1` 驗證 `get_financial_metrics()` 能提供現有分析代理常用欄位
- [ ] `P1` 驗證 `get_market_cap()` 在當日與歷史日期都能拿到合理值
- [ ] `P1` 驗證 `get_insider_trades()` 的日期範圍與 pagination 不退化
- [ ] `P1` 驗證 `search_line_items()` 能滿足現有 agents 所需欄位
- [ ] `P1` 驗證 backtesting / integration tests 不因 provider 切換而失敗

## Phase 4 / 快取與可觀測性

### Financial Data Cache

- [ ] `P2` 為金融資料查詢加入本地快取機制
- [ ] `P2` 定義 cache keys、TTL、失效策略與持久化格式
- [ ] `P2` 確保相同請求會重用快取，而不是重複打上游 provider
- [ ] `P2` 加入 cache hit rate、miss、stale reads、upstream request count 觀測能力
- [ ] `P2` 驗證快取資料與 fresh upstream response 的一致性
- [ ] `P2` 測試 repeated queries、partial overlaps、missing fields、expired entries

### Expert Output Cache

- [ ] `P2` 為專家意見加入本地快取機制
- [ ] `P2` 以輸入資料 fingerprint、專家 identity 與 model settings 定義 cache key
- [ ] `P2` 確保相同資料加上相同專家時會重用輸出，而不是重複呼叫 model
- [ ] `P2` 驗證 prompt、model version、input schema 變更時快取失效邏輯正確

## Phase 5 / Expert System 優化

### Weighting and Evaluation

- [ ] `P2` 加入可配置的 expert-weighting 機制，支援 equal weight 與 custom weights
- [ ] `P2` 回測 `src/agents/sentiment.py` 中 insider trades `0.3` / news sentiment `0.7` 權重是否合理，並比較其他配比
- [ ] `P2` 研究 expert aggregation 是否應維持等權重，或透過 backtesting 找出更佳權重
- [ ] `P2` 回測等權重與其他權重方案，衡量風險調整後績效
- [ ] `P2` 驗證權重改善是否能跨 symbols、sectors、market regimes 穩定成立

### Expert Framework

- [ ] `P3` 研究如何評估 expert opinions 的有效性、可信度與真實性
- [ ] `P3` 建立新增與版本化 expert 的框架
- [ ] `P3` 研究如何建立具備不同風格與前提的新 experts
- [ ] `P3` 驗證新 expert 是否真的具備差異性，而不是 prompt 變體
- [ ] `P3` 蒐集可參考的方法論與 persona，例如不同技術分析、基本面與估值框架

## Later Research / 較後期研究

- [ ] `P3` 比較更多替代資料來源的 coverage、latency、reliability、rate limits 與 licensing
- [ ] `P3` 評估自建資料管線是否可行
- [ ] `P3` 若考慮自建，再評估 exchange APIs、public filings、bulk downloads、scheduled crawlers、local databases 等方案
- [ ] `P3` 在真正採用自建方案前，先確認維護成本與長期可行性

## Definition of Done / 完成標準

- [ ] `P1` 現有 `financialdatasets.ai` 的 `prices` / `news` 不再因 request 格式不符而觸發 `HTTP 400`
- [ ] `P1` provider layer 上線後，agents/backtest 不需改呼叫介面即可切換資料來源
- [ ] `P1` FMP 能覆蓋目前主流程需要的價格、新聞、財務指標、市值、內線交易與必要 line items
- [ ] `P2` 快取與測試補齊後，回測與 agent 執行成本、穩定性與可追查性明顯改善
