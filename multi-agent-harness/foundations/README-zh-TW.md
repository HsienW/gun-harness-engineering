# Multi-Agent Harness 基礎篇

一封交辦信要讓 Worker 知道在哪個工作區行動、向誰回覆，也要讓接手驗收的人認得交付物。以下以 Munder Difflin 的 Hive 協作協定為例。

1. [如何把工作交給另一個 Agent？](./01-delegation-contract-zh-TW.md)　從一個 README 查核任務，定出收件人、工作範圍與回報條件。
2. [收到回信後，誰能判定交付完成？](./02-delivery-and-acceptance-zh-TW.md)　把實作及驗證的結果對回同一個版本，決定是否需要補件。
3. [一件工作該拆給哪些 Worker？](./03-work-decomposition-zh-TW.md)　回到派工前，判斷哪些工作可並行、誰需要等上游交付，以及何時一位 Worker 就夠。
4. [要求改了，哪些委派還有效？](./04-revising-delegated-work-zh-TW.md)　沿著一次範圍更正，核對受影響的交辦、可沿用的產物與需要補測的版本。

信件與回覆關聯參考 Munder Difflin 的固定版本。各篇的任務分工、版本 B 與 C、改派及驗收判斷均為教學情境。[版本 96ae43e](https://github.com/HarnessMD/munder-difflin/commit/96ae43e3292447f68af6904afb353f6f5bdaa412) · [Hive 協作協定](https://github.com/HarnessMD/munder-difflin/blob/96ae43e3292447f68af6904afb353f6f5bdaa412/HIVE.md)

[回到 Multi-Agent Harness](../README-zh-TW.md)
