# Multi-Agent Runtime 基礎篇

Munder Difflin 主程序透過虛擬終端（PTY）啟動 Worker CLI。Hive 管理交辦和信件，模型取樣與工具結果回灌則在 CLI 的回合裡進行。追查一次執行時，可以先從主程序持有的身分找到 Worker，再沿著信件找待處理的工作。[平台架構](https://github.com/HarnessMD/munder-difflin/blob/96ae43e3292447f68af6904afb353f6f5bdaa412/docs/ARCHITECTURE.md)

1. [Worker 重啟後，平台還在跟誰說話？](./01-worker-identity-zh-TW.md)　辨認五種 ID、同一個 PTY 下的替代程序，以及啟動參數遺失的故障。
2. [信已送達，Worker 何時會處理？](./02-mailbox-and-cli-progress-zh-TW.md)　分開觀察 Router 投遞、CLI 收件、回合停止與回信。
3. [平台重啟後，憑什麼接回原任務？](./03-recovery-and-state-evidence-zh-TW.md)　查角色記憶、Transcript、CLI Session 和 Task，判斷還需要哪些證據。

原始碼、設計文件與修復紀錄固定在 [96ae43e](https://github.com/HarnessMD/munder-difflin/commit/96ae43e3292447f68af6904afb353f6f5bdaa412)。PTY 回呼及 Session 登錄可查原始碼，Stop hook 的預期行為來自 Hive 設計文件。各篇標出的演練仍待實測。

[回到 Multi-Agent Runtime](../README-zh-TW.md)
