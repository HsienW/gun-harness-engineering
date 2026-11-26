# 如何把工作交給另一個 Agent？

[English](./01-delegation-contract.md) | [繁體中文](./01-delegation-contract-zh-TW.md) | [简体中文](./01-delegation-contract-zh-CN.md)

假設協調者要查 README 裡的一個失效連結。只交代「檢查文件」，Worker 即使回覆「已檢查」，接手的人仍得猜它查了哪個版本。把工作縮到指定工作區、版本和檔案，回信附行號與目標路徑，另一個人就能重做同一次查核。

| 交辦時要說清楚什麼 | 這次查核的約定 |
|---|---|
| 誰交給誰 | 協調者建立 Task、指定負責的 Worker，留存原交辦的 Message ID。 |
| 可以做什麼 | Worker 讀指定版本的 README，依文件所在目錄解析相對路徑。若需要修改檔案，先回報，由協調者確認是否擴大範圍。 |
| 怎麼回報 | Worker 透過 outbox 回信，用 `in_reply_to` 指向原件，附上版本、行號和失效路徑。 |
| 誰結案 | 協調者在指定版本開啟檔案、核對目標，再把驗收結果登記到 Task。 |

## Worker 收到協作規則了嗎？

Worker 需要取得 inbox、outbox 的使用方式，才能讀取交辦並對原件回覆。Munder Difflin 曾出現 Windows 啟動成功、Worker 卻少了這段指令的情況。可從協調者送出一封有識別碼的請求，核對 Worker 的回信是否指向原件。啟動參數截斷的位置與程序端證據見 [Worker 啟動與身分](../../multi-agent-runtime/foundations/01-worker-identity-zh-TW.md)。

Hive 設計區分需要回答的請求與可結束往返的通知。協調者要更換收件人或改查另一個版本時，也應在交辦中指出新的負責人與目標，讓驗收者知道哪份回報仍適用。上表是教學任務的約定，實際訊息欄位見 [Hive 協作協定](https://github.com/HarnessMD/munder-difflin/blob/96ae43e3292447f68af6904afb353f6f5bdaa412/HIVE.md)。

[回到本區基礎篇](./README-zh-TW.md)
