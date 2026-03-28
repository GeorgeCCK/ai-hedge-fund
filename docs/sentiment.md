# Sentiment Analysts Overview

這份文件整合兩條情緒分析路徑：

- `src/agents/sentiment.py`
- `src/agents/news_sentiment.py`

重點不是只看單一檔案，而是從「前面怎麼選 analyst」開始，一路說明：

- 使用者怎麼選到這兩個 analyst
- workflow 怎麼把它們接進 graph
- 兩個 agent 各自怎麼運作
- 兩者的資料依賴、輸出與限制差在哪裡

## 從哪裡選到這兩個 Analyst

這兩個 analyst 都定義在 `src/utils/analysts.py` 的 `ANALYST_CONFIG`：

- `news_sentiment_analyst` -> `news_sentiment_agent`
- `sentiment_analyst` -> `sentiment_analyst_agent`

也就是說，從系統角度看，前端、CLI、backend graph 都不是直接引用檔名，而是先引用 analyst key，再映射到真正的 agent function。

## 進入點 1: CLI

CLI 入口在 `src/cli/input.py`。

使用者有三種方式選 analyst：

- 用 `--analysts` 明確指定
- 用 `--analysts-all` 全選
- 不帶參數時，用互動式 checkbox 選擇

這份選擇清單來自 `ANALYST_ORDER`，而 `ANALYST_ORDER` 又是從 `ANALYST_CONFIG` 推導出來，所以 `news_sentiment_analyst` 和 `sentiment_analyst` 都會出現在 CLI 選單中。

## 進入點 2: 預設 workflow

在 `src/main.py` 中，`create_workflow(selected_analysts)` 會根據傳入的 analyst keys 建 graph。

流程是：

1. 先從 `get_analyst_nodes()` 取得 `analyst_key -> (node_name, agent_func)` 映射
2. 把選中的 analyst 加入 graph node
3. 將 `start_node` 接到每個 analyst
4. 再把 analyst 接到 risk manager
5. 最後進 portfolio manager

所以如果使用者選了：

- `news_sentiment_analyst`
- `sentiment_analyst`

那 workflow 就會同時跑兩條不同的情緒分析節點，最後都把結果寫進 `state["data"]["analyst_signals"]`。

## 進入點 3: Backend / Frontend Graph

在 web app 路徑中，`app/backend/routes/hedge_fund.py` 會接收前端送來的 graph 結構，然後交給 `app/backend/services/graph.py:create_graph()`。

這條路徑的特點是：

- 前端送的是 graph nodes / edges
- backend 先從節點 ID 萃取 base agent key
- 再用 `ANALYST_CONFIG` 找到對應 agent function

也就是說，在前端 flow editor 中放入 `news_sentiment_analyst` 或 `sentiment_analyst` 節點時，後端仍然是透過同一份 analyst config 找到真正要執行的函式。

## 共同資料流

不管是 `sentiment_analyst_agent()` 還是 `news_sentiment_agent()`，它們都吃相似的 state 結構：

- `data.tickers`
- `data.end_date`
- `metadata.show_reasoning`
- `FINANCIAL_DATASETS_API_KEY`

兩者都會：

- 逐一處理每個 ticker
- 產生 ticker 層級的 `signal` / `confidence` / `reasoning`
- 把結果寫入 `state["data"]["analyst_signals"][agent_id]`
- 回傳 `HumanMessage`

差異在於：它們如何產生情緒訊號。

## 路徑 A: `news_sentiment.py`

`src/agents/news_sentiment.py` 的定位是純新聞情緒分析器。

### 它做什麼

對每個 ticker：

1. 呼叫 `get_company_news()`
2. 取最近 10 則新聞
3. 找出缺少 `sentiment` 的新聞
4. 最多挑 5 則，用 LLM 根據標題補做分類
5. 把所有有 sentiment 的新聞轉成 `bullish` / `bearish` / `neutral`
6. 用多數決決定最終方向

### LLM 補分類

這支 agent 最重要的特性，是它不完全依賴新聞 provider 已經帶好 `sentiment`。

如果 `news.sentiment is None`，它會呼叫 `call_llm(...)`，要求模型輸出：

- `sentiment`: `positive` / `negative` / `neutral`
- `confidence`: 0 到 100

如果 LLM 失敗，會回退成：

- `sentiment = "neutral"`
- `confidence = 0`

### 最終判定

新聞的映射規則是：

- `positive` -> `bullish`
- `negative` -> `bearish`
- 其他 -> `neutral`

接著直接統計數量：

- bullish 比 bearish 多 -> `bullish`
- bearish 比 bullish 多 -> `bearish`
- 相等 -> `neutral`

這裡沒有內線交易，也沒有額外來源權重。

### Confidence

如果這次有由 LLM 補分類的新聞，confidence 會混合：

- 70% LLM confidence
- 30% signal proportion

否則就退回單純比例法。

### 輸出重點

`reasoning` 只有一個主區塊：

- `news_sentiment`

其中會記錄：

- 多空中性文章數
- 有多少篇是靠 LLM 補分類

## 路徑 B: `sentiment.py`

`src/agents/sentiment.py` 的定位是混合型市場情緒分析器。

### 它做什麼

對每個 ticker：

1. 呼叫 `get_insider_trades()`
2. 依 `transaction_shares` 正負，把每筆交易轉成多空
3. 呼叫 `get_company_news()`
4. 直接讀每篇新聞的 `sentiment`
5. 把 insider 與 news 做加權整合
6. 產生最終 `signal`、`confidence` 與 `reasoning`

### Insider trades 如何轉訊號

規則很直接：

- `transaction_shares < 0` -> `bearish`
- `transaction_shares >= 0` -> `bullish`

它不看交易品質、交易金額或 insider 身分，只看股數正負。

### 新聞如何轉訊號

它會直接使用 provider 已給的 `news.sentiment`：

- `positive` -> `bullish`
- `negative` -> `bearish`
- 其他 -> `neutral`

這裡沒有 LLM fallback。

如果新聞沒有 `sentiment`，這些項目在 `dropna()` 後不會留下有效訊號。

### 加權方式

它固定使用：

- insider trades: `0.3`
- news sentiment: `0.7`

這不是逐筆比對，而是各自先統計 bullish / bearish 數量，再乘上權重。

### Confidence

這支 agent 的 confidence 比較簡單，等於：

```text
max(weighted_bullish, weighted_bearish) / total_weighted_signals * 100
```

如果沒有有效訊號，confidence 是 `0`。

### 輸出重點

`reasoning` 會分三塊：

- `insider_trading`
- `news_sentiment`
- `combined_analysis`

所以它比 `news_sentiment.py` 多了來源拆解與加權後總結。

## 兩者對照

| 面向 | `news_sentiment.py` | `sentiment.py` |
| --- | --- | --- |
| Analyst key | `news_sentiment_analyst` | `sentiment_analyst` |
| 核心資料來源 | 只有 company news | insider trades + company news |
| 對 `news.sentiment` 的依賴 | 可缺值，會用 LLM 補 | 強依賴 provider 已提供 |
| 是否用 LLM | 會，最多補最近 10 則中的 5 則缺值新聞 | 不會 |
| 新聞統計方式 | 每篇新聞等權重 | 新聞先等權重計數，再與 insider 做 `0.7 / 0.3` 加權 |
| 是否使用 insider trades | 否 | 是 |
| 最終 signal | 新聞多數決 | insider 與新聞加權後比較 |
| confidence | LLM confidence 與 signal proportion 混合，否則退回比例法 | 加權後優勢方向占全部加權訊號的比例 |
| reasoning 結構 | 單一 `news_sentiment` 區塊 | `insider_trading`、`news_sentiment`、`combined_analysis` |
| 適合的資料條件 | provider 沒有 sentiment 欄位時仍可運作 | provider 有完整 sentiment 欄位時較合理 |

## 什麼情況下兩者結果會明顯不同

### 情況 1: provider 回傳的新聞沒有 `sentiment`

- `news_sentiment.py` 仍可運作，因為它會補做分類
- `sentiment.py` 不會報錯，但新聞影響會被低估

這是目前兩者最重要的差別。

### 情況 2: insider trades 和新聞方向相反

例如：

- insider 偏多
- 新聞偏空

這時：

- `news_sentiment.py` 只會反映新聞偏空
- `sentiment.py` 會做綜合判斷，而且新聞權重比較高

### 情況 3: 新聞量很多但只有少部分有 sentiment

- `news_sentiment.py` 至少會嘗試補最近的一部分
- `sentiment.py` 只會吃到原本就有 sentiment 的那部分

## 目前設計上的限制

兩者都有簡化假設，但限制點不太一樣。

### `news_sentiment.py` 的限制

1. 只看最近 10 則新聞
2. 最多只補 5 則缺值新聞
3. 只看標題，不看文章全文
4. 每篇新聞等權重
5. confidence 部分依賴 LLM 自報分數

### `sentiment.py` 的限制

1. 強依賴 provider 已提供 `news.sentiment`
2. 沒有 LLM fallback
3. insider trades 只看股數正負
4. 權重 `0.3 / 0.7` 是固定常數
5. 新聞與 insider 都只按筆數計數

## 實務上的理解方式

可以把這兩支 agent 想成兩條不同的情緒分析產品線：

- `news_sentiment_analyst`: 新聞優先、欄位缺失時可補救
- `sentiment_analyst`: 多來源整合，但要求上游資料更完整

如果資料源經常只回傳標題、來源、日期、URL，而沒有 `sentiment`，那 `news_sentiment_analyst` 目前會比 `sentiment_analyst` 更可靠。

如果未來資料源穩定提供高品質的新聞 sentiment，而且想把 insider behavior 一起考慮進來，`sentiment_analyst` 才比較能發揮它的設計目的。
