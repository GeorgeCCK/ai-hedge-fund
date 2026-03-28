# TODO-FIN-SRC-1

## Summary
目標是替這個專案找出 `financialdatasets.ai` 的低成本替代方案，並規劃可實作的遷移路線。
目前專案不只新聞代理依賴資料源，還包含價格、財務指標、line items、內線交易與 company facts，因此替代方案必須以整體資料面評估，而不是只看 `news_sentiment`。

目前建議的優先順序：

1. FMP (`Financial Modeling Prep`) 作為主替代方案
2. Alpha Vantage 作為超低預算備選
3. EODHD 作為全球市場導向備選
4. Polygon 暫不列為低成本優先方案

## Current State
目前 `financialdatasets.ai` 依賴集中在：
- `src/tools/api.py`
- `src/agents/news_sentiment.py`
- `app/frontend/src/components/settings/api-keys.tsx`

目前 repo 使用到的資料能力包含：
- 歷史價格
- financial metrics
- line items 搜尋
- insider trades
- company news
- company facts / market cap

目前 `news_sentiment` 只依賴標準化後的 `CompanyNews` 結構：
- `ticker`
- `title`
- `source`
- `date`
- `url`
- `sentiment`

這表示新聞代理本身不一定需要大改，重點在 data provider adapter。

## Recommended Provider
### 1. FMP
最接近單一供應商替代，成本與覆蓋率平衡最好。
適合先當第一個落地版本。

優點：
- 有歷史價格
- 有新聞
- 有 key metrics / ratios
- 有 company profile
- 有 insider trading
- 遷移時最容易維持目前 repo 架構

缺點：
- `search_line_items` 沒有完全等價接口，需要 statement mapping

官方參考：
- https://site.financialmodelingprep.com/developer/docs/stable
- https://site.financialmodelingprep.com/developer/docs/stable/search-stock-news/
- https://site.financialmodelingprep.com/developer/docs/stable/search-insider-trades
- https://site.financialmodelingprep.com/developer/docs/stable/key-metrics
- https://site.financialmodelingprep.com/developer/docs/stable/metrics-ratios
- https://site.financialmodelingprep.com/developer/docs/stable/profile-symbol
- https://site.financialmodelingprep.com/pricing-plans

### 2. Alpha Vantage
適合極低預算與 demo 使用，但不適合作為這個 repo 的最佳主供應商。

優點：
- 有 `NEWS_SENTIMENT`
- 有公司概覽與財報接口
- 初期導入門檻低

缺點：
- 免費額度偏小
- premium gating 對回測與批量查詢不友善
- 欄位映射與節流限制會增加整合成本

官方參考：
- https://www.alphavantage.co/documentation/
- https://www.alphavantage.co/premium/
- https://www.alphavantage.co/support/

### 3. EODHD
適合之後想擴到全球市場時使用。

優點：
- 全球市場覆蓋廣
- 有 EOD 價格
- 有 fundamentals
- 有 financial news
- 有 insider transactions

缺點：
- 若只為了替掉目前 repo 的美股主流程，成本效益通常不如 FMP

官方參考：
- https://eodhd.com/financial-apis
- https://eodhd.com/lp/fundamental-data-api
- https://eodhd.com/pricing

### 4. Polygon
資料品質佳，但不符合「最低成本優先」。

優點：
- 美股價格與新聞能力強
- reference / ticker overview 清楚

缺點：
- financials / deeper fundamentals 通常落在較高 tier
- 總成本不適合這次方向

官方參考：
- https://polygon.io/docs/stocks
- https://polygon.io/docs/rest/stocks/news/
- https://polygon.io/docs/rest/stocks/tickers/ticker-overview/
- https://polygon.io/docs/rest/stocks/fundamentals/financials
- https://polygon.io/pricing/

## Implementation Plan
### Phase 1: 抽象資料供應商層
- [ ] 在 `src/tools/api.py` 前加一層 provider dispatch
- [ ] 讓 agents 繼續呼叫既有函式，不直接感知供應商
- [ ] 新增 `FINANCIAL_DATA_PROVIDER` 環境變數
- [ ] 預設 provider 設為 `fmp`

### Phase 2: 導入 FMP
先完成以下函式的 FMP 版本：
- [ ] `get_prices`
- [ ] `get_company_news`
- [ ] `get_financial_metrics`
- [ ] `get_insider_trades`
- [ ] `get_market_cap`

要求：
- [ ] 對上層輸出 shape 保持不變
- [ ] adapter 內處理欄位名稱差異
- [ ] 盡量沿用現有 cache 行為

### Phase 3: 重做 line item 取得方式
- [ ] 保留 `search_line_items()` 對上層的函式介面
- [ ] 底層改用 FMP 財報資料映射成 `LineItem`
- [ ] 不追求和 `financialdatasets.ai` API 100% 對等
- [ ] 只覆蓋目前 agents 會用到的關鍵欄位

### Phase 4: 更新設定與文件
- [ ] 前端 API key 設定增加 `FMP_API_KEY`
- [ ] 文件說明改為支援 provider 切換
- [ ] 後端讀 key 邏輯支援新 provider
- [ ] 保留未來增加 `ALPHA_VANTAGE_API_KEY` / `EODHD_API_KEY` 的空間

## Interfaces / Config
新增：
- `FINANCIAL_DATA_PROVIDER=fmp`
- `FMP_API_KEY=...`

保留：
- agents 對 `src.tools.api` 的呼叫方式不變
- `CompanyNews`
- `FinancialMetrics`
- `Price`
- `InsiderTrade`
- `LineItem`

## Test Plan
必測案例：
- [ ] `get_company_news()` 能映射出正確的 `CompanyNews`
- [ ] `news_sentiment_agent()` 在缺少 sentiment 時仍能走既有 LLM fallback
- [ ] `get_prices()` 的 OHLCV 與時間排序符合目前 DataFrame 轉換假設
- [ ] `get_financial_metrics()` 能提供現有分析代理常用欄位
- [ ] `get_market_cap()` 在當日與歷史日期都能拿到合理值
- [ ] `get_insider_trades()` 的日期範圍與 pagination 不退化
- [ ] `search_line_items()` 能滿足現有 agents 所需欄位
- [ ] backtesting / integration tests 不因 provider 切換而失敗

## Assumptions
- 目前主要目標市場是美股
- 這次優先考量成本，而不是最高品質或最低延遲
- 第一階段接受只先支援 FMP
- `search_line_items()` 可接受用 statement mapping 取代原生搜尋 API
