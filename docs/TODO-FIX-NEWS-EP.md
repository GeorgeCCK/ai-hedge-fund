# TODO-FIX-NEWS-EP

## Summary

修正 Financial Datasets 新聞 API 整合，讓 `/news` 請求符合目前官方文件，避免 `HTTP 400 Bad Request`，並讓既有 agent/backtest 在功能降級下仍可正常運作。

參考文件：
- https://docs.financialdatasets.ai/api/news/company

## Problem

目前 `get_company_news()` 的實作和官方文件存在落差，可能導致 `HTTP 400 Bad Request`：

- 程式目前呼叫 `https://api.financialdatasets.ai/news/?ticker=...&end_date=...`
- 若有 `start_date` 也會一併送出
- `limit` 可能被設成 `50`、`100`、`250`、`1000`

但官方文件目前明列：

- endpoint: `GET https://api.financialdatasets.ai/news`
- 必填 query param: `ticker`
- 可選 query param: `limit`
- `limit` 最大值：`10`

## Tasks

- 更新 `src/tools/api.py` 中 `get_company_news()` 的 request 組裝邏輯
- 將 endpoint 統一為 `https://api.financialdatasets.ai/news`
- 只送官方文件明列的 query params
  - `ticker`
  - `limit`
- 將 `limit` 超過 `10` 的請求自動壓到 `10`
- 停止送出 `start_date`、`end_date`
- 保留現有函式簽名，避免上層呼叫全面改寫
- 對不支援的歷史篩選或過大 `limit` 加上 warning log
- 檢查所有 `get_company_news(..., limit=...)` 呼叫點，確認降級後不會造成例外
- 視需要調整 cache key，避免使用已忽略參數造成誤導性快取

## Impacted Areas

- `src/tools/api.py`
- `src/backtesting/engine.py`
- `app/backend/services/backtest_service.py`
- 使用 `get_company_news()` 的 agent 模組

## Acceptance Criteria

- `/news` API 請求不再因不支援的 query params 而觸發 `HTTP 400`
- `get_company_news()` 發出的 request 只包含文件允許的參數
- `limit > 10` 時會自動 clamp 到 `10`
- backtest 在新聞資料降級為最多 `10` 筆時仍可正常執行
- 新聞相關 agent 在資料筆數變少時不會拋出例外
- 若呼叫端仍傳入 `start_date` 或大於 `10` 的 `limit`，log 中可看出降級行為

## Tests

- 為 `get_company_news()` 補上 request 組裝測試
- 驗證不再送出 `start_date`、`end_date`
- 驗證 `limit` 會被 clamp 到 `10`
- 驗證 API 成功回傳時 `CompanyNewsResponse` 仍可正常 parse
- 驗證 backtest 路徑在最多只拿到 `10` 筆新聞時仍可完成執行
- 驗證 sentiment/news 相關 agent 在低樣本新聞下不會報錯

## Notes

- 目前沒有證據顯示 repo 內有本地 `GET /news` route；400 問題較可能來自外部 Financial Datasets API 請求格式不符。
- 這次修正先以「遵循目前官方文件」為原則，不假設未文件化的歷史新聞查詢能力仍可用。
