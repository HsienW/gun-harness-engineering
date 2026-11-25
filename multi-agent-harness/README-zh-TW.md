# Multi-Agent Harness

協調者把修正與測試分給兩位 Worker。實作者交出版本 B，驗證者回信說「測試通過」。這時有兩份回報，協調者還要知道測試用的是哪一版，才能向使用者交付。平台 Harness 負責訂出這類協作規則，包括誰接手、誰可以更正要求，以及什麼證據足以結案。

[Foundations 基礎篇](./foundations/README-zh-TW.md) 從一封交辦信寫到驗收，再討論如何分工與更正要求。追查信件何時送達、Worker 重啟後由誰接收，可讀 [Multi-Agent Runtime](../multi-agent-runtime/README-zh-TW.md)。

## 平台協作專章

| 類目 | 文章 |
|---|---|
| Input and Context | [另一位 Worker 的回信，能改變我的任務嗎？](./input-and-context/01-peer-replies-and-task-scope-zh-TW.md) |
| Behavior and Capabilities | [Worker 遇到阻礙時，該追問、轉交，還是回報？](./behavior-and-capabilities/01-query-handoff-and-escalation-zh-TW.md) |
| Action Governance | [A 獲准的操作，改派給 B 後還有效嗎？](./action-governance/01-approval-after-reassignment-zh-TW.md) |
| Interaction and Feedback | [使用者說「先停」，平台能承諾停到哪裡？](./interaction-and-feedback/01-stop-across-workers-zh-TW.md) |

## 實作參照

* [munder-difflin](https://github.com/HarnessMD/munder-difflin)
