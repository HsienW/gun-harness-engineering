# Multi-Agent Runtime

Worker 重啟後仍出現在同一個終端視窗，舊程序的退出事件卻可能這時才到。要保住新程序的輸入通道，主程序得認出回呼屬於哪個程序實例。若還要接回上一件工作，就得再查原 Task、信件及 CLI 對話。

[Foundations 基礎篇](./foundations/README-zh-TW.md) 從程序身分開始，接著追信件投遞、CLI 回合與重啟恢復。委派的授權、範圍與交付條件見 [Multi-Agent Harness](../multi-agent-harness/README-zh-TW.md)。

## 實作參照

* [munder-difflin](https://github.com/HarnessMD/munder-difflin)
