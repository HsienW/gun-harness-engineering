# Agent Runtime

[English](./README.md) | [繁體中文](./README-zh-TW.md) | [简体中文](./README-zh-CN.md)

本區整理 Agent 的執行核心：一次 Run 如何取得執行身分、推進狀態、呼叫模型與工具、等待外部決策、處理副作用與故障，最後產生可驗證的終止結果。

Runtime 接收 [Agent Harness](../agent-harness/README-zh-TW.md) 準備好的輸入、Context、能力與政策條件，但不重新決定這些產品規則。執行途中若需要授權、人工確認或使用者補充，Runtime 透過明確的 Decision Port 把問題交還 Harness，再依結構化決策繼續或停止。

## 閱讀架構

| 分類 | 範圍 |
|---|---|
| [Foundations](./foundations/README-zh-TW.md) | Runtime 定義、Execution Identity 與狀態權威、Run Loop、持久化與恢復、設計原則及一次完整 Run。 |
| [Execution Lifecycle](./execution-lifecycle/README-zh-TW.md) | Run 接納與建立、執行歸屬與替換、期限與取消、等待恢復、終止清理及生命週期競態。 |
| [Model and Tool Execution](./model-and-tool-execution/README-zh-TW.md) | 模型呼叫與 Provider 切換、串流組裝、工具派送與並行排程、背壓容量、結果回灌及一次模型與工具往返。 |
| [Effects and Recovery](./effects-and-recovery/README-zh-TW.md) | Checkpoint 與恢復資料、冪等與重放識別、效果帳本與提交邊界、回查重試、補償及提交不明的恢復案例。 |
| [Events and Operations](./events-and-operations/README-zh-TW.md) | 事件身分與順序、保存交付、追蹤稽核與資料邊界、指標健康、故障驗證與上線門檻及漏接終止事件的查證。 |

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

## 證據與適用範圍

本區沿用全庫的 Engineering 標記，分開描述已觀察的原始碼、已接入正式入口的能力、只存在於模組或測試中的 Primitive，以及仍待實作或部署驗證的設計。類別、測試案例、記憶體 Checkpointer 或成功回覆都不能單獨證明跨程序恢復、外部副作用冪等或正式環境可用。
