# TODO-FIX-PRICES-EP

## Summary

修正 Financial Datasets 歷史價格 API 整合，讓 `/prices` 請求符合目前官方文件，避免 `HTTP 400 Bad Request`，並確保既有 agent/backtest 仍可正常取得日線價格資料。

參考文件：
- https://docs.financialdatasets.ai/api/prices/historical
- https://docs.financialdatasets.ai/api/prices/snapshot

## Problem

目前 `get_prices()` 的實作和官方文件存在落差，可能導致 `HTTP 400 Bad Request`：

- 程式目前呼叫 `https://api.financialdatasets.ai/prices/?ticker=...&interval=day&interval_multiplier=1&start_date=...&end_date=...`
- 固定送出 `interval=day`
- 額外送出 `interval_multiplier=1`

但官方文件目前明列：

- historical endpoint: `GET https://api.financialdatasets.ai/prices`
- 必填 query param:
  - `ticker`
  - `interval`
  - `start_date`
  - `end_date`
- 文件中沒有 `interval_multiplier`

另外，snapshot 文件顯示：

- snapshot endpoint: `GET https://api.financialdatasets.ai/prices/snapshot`
- 必填 query param: `ticker`

目前 repo 內沒有看到實際使用 snapshot endpoint，因此這次 `400` 問題較可能來自 historical `/prices` 的 request 格式不符。

## Tasks

- 更新 `src/tools/api.py` 中 `get_prices()` 的 request 組裝邏輯
- 將 historical prices endpoint 統一為 `https://api.financialdatasets.ai/prices`
- 只送官方文件明列的 query params
  - `ticker`
  - `interval`
  - `start_date`
  - `end_date`
- 停止送出 `interval_multiplier`
- 保留現有函式簽名，避免上層呼叫全面改寫
- 維持目前 `interval=day` 的既有行為，避免改變 agent/backtest 語意
- 檢查所有 `get_prices()` 呼叫點，確認移除 `interval_multiplier` 後不會造成其他相依邏輯問題
- 視需要補上 log，讓非 200 回應更容易追查 request 問題
- 確認 `PriceResponse` 仍可正確 parse 最新 API 回傳格式

## Impacted Areas

- `src/tools/api.py`
- `src/agents/technicals.py`
- `src/agents/risk_manager.py`
- `src/agents/stanley_druckenmiller.py`
- `src/agents/nassim_taleb.py`
- `src/backtesting/engine.py`
- `app/backend/services/backtest_service.py`

## Acceptance Criteria

- `/prices` API 請求不再因不支援的 query params 而觸發 `HTTP 400`
- `get_prices()` 發出的 request 只包含文件允許的參數
- `interval_multiplier` 不再出現在 historical prices request 中
- 既有使用 `get_prices()` 的 agent 與 backtest 不需要修改呼叫方式即可正常運作
- API 成功回傳時，價格資料仍可被 `PriceResponse` 正常 parse
- 價格資料轉 DataFrame 的流程不受影響

## Tests

- 為 `get_prices()` 補上 request 組裝測試
- 驗證不再送出 `interval_multiplier`
- 驗證 request 仍包含 `ticker`、`interval`、`start_date`、`end_date`
- 驗證 API 成功回傳時 `PriceResponse` 仍可正常 parse
- 驗證既有 rate limit retry 行為不受影響
- 驗證至少一條會經過 `get_prices()` 的 agent 或 backtest 路徑可正常完成

## Notes

- 這次修正先以「遵循目前官方文件」為原則，不假設 `interval_multiplier` 仍被後端接受。
- 雖然也檢查了 snapshot 文件，但目前沒有證據顯示 repo 內有實際使用 `GET /prices/snapshot`，因此本次修正先聚焦 historical `/prices`。
- `Price` model 目前未包含 `time_milliseconds`，但只要 Pydantic 維持忽略額外欄位，現有 parse 應可相容。
