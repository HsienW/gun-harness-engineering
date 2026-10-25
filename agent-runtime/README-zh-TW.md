# Agent Runtime

[English](./README.md) | [繁體中文](./README-zh-TW.md)

本區整理 Agent 的執行核心：一次 Run 如何取得執行身分、推進狀態、呼叫模型與工具、等待外部決策、處理副作用與故障，最後產生可驗證的終止結果。

Runtime 接收 [Agent Harness](../agent-harness/README-zh-TW.md) 準備好的輸入、Context、能力與政策條件，但不重新決定這些產品規則。執行途中若需要授權、人工確認或使用者補充，Runtime 透過明確的 Decision Port 把問題交還 Harness，再依結構化決策繼續或停止。

## 閱讀架構

| 分類 | 範圍 | 狀態 |
|---|---|---|
| [Foundations](./foundations/README-zh-TW.md) | Runtime 定義、Execution Identity、狀態轉移、持久化與恢復、設計原則及一次完整 Run。 | 六篇正體中文初稿完成 |
| `execution-lifecycle/` | Admission、Run／Task／Thread、期限、取消、等待、終止與生命週期競態。 | 規劃中 |
| `model-and-tool-execution/` | 模型串流、Tool Call 迴圈、排程、並行、錯誤回灌、資源與成本預算。 | 規劃中 |
| `effects-and-recovery/` | Checkpoint、Interrupt、Resume、Idempotency、Side-effect Ledger、Reconciliation 與 Compensation。 | 規劃中 |
| `events-and-operations/` | Runtime Event、Tracing、Metrics、Audit、故障演練與上線門檻。 | 規劃中 |

## Runtime 的責任位置

~~~text
Harness 交付 Execution Request
  → Runtime 驗證執行契約並建立 Run Identity
  → Runtime 推進模型、工具、等待與恢復狀態
  → Runtime 回報事件、證據與 Terminal Result
  → Harness 將結果轉成產品互動
~~~

這條流程描述責任，不代表所有實作都使用相同類別或同一套狀態列舉。Imperative query loop、StateGraph 與遠端 Agent Server 都可以承擔 Runtime；判斷依據是誰持有執行狀態、合法轉移、停止條件與恢復證據。

## 與 Harness 的分工

| 問題 | 主要權威 |
|---|---|
| 使用者輸入如何正規化、哪些 Context 可見 | Harness |
| 能力集合、授權政策與人工確認規則 | Harness |
| Run 如何開始、等待、取消及收斂 | Runtime |
| 模型與工具如何依預算推進 | Runtime |
| 外部操作是否真正提交 | 外部系統與 Side-effect Evidence |
| 執行結果如何呈現給使用者 | Harness |

Runtime 可以執行 Harness 已交付的規則，也可以在缺少決策時停止並請求外部判斷；它不能從 Prompt、工具參數或 UI 文案自行創造可信身分、授權或外部提交事實。

## 證據寫法

本區沿用全庫的 Engineering 標記，分開描述已觀察的原始碼、已接入正式入口的能力、只存在於模組或測試中的 Primitive，以及仍待實作或部署驗證的設計。類別、測試案例、記憶體 Checkpointer 或成功回覆都不能單獨證明跨程序恢復、外部副作用冪等或正式環境可用。
