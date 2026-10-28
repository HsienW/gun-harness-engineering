# Checkpoint 與恢復資料

[English](./01-checkpoints-and-recovery-state.md) | [繁體中文](./01-checkpoints-and-recovery-state-zh-TW.md)

> 以下內容所提到的
> CCB 為 [claude-code-best](https://github.com/claude-code-best/claude-code) 的簡稱。
> Chat Gun 為 [chat-gun](https://github.com/HsienW/chat-gun)。

[回到導覽](./README-zh-TW.md) · 下一篇：[冪等與重放識別](./02-idempotency-and-replay-identity-zh-TW.md)

> 程序停止前，工具已經送出寫入；重啟後讀回的 Checkpoint 卻仍顯示這個節點尚待執行。若直接從節點位置重跑，外部系統可能收到第二次操作。若只恢復畫面上的對話，Runtime 又可能遺失尚待確認的操作與版本條件。

**恢復需要交叉核對執行位置、等待契約與外部效果證據。** Checkpoint 提供其中一份狀態，是否可以續行，還要沿著 Manifest、Task／Step 與 Ledger 查明。

## 需要讀回哪些資料

Graph 可以記得下一個節點，Task repository 可以記得工作仍在 running，外部系統則可能早已提交。這些資料記錄不同事實，程序停止的位置會讓它們暫時不一致。

| 資料 | 主要用途 | 無法單獨回答的問題 |
|---|---|---|
| Checkpoint | 保存 Graph values、訊息與待處理節點。 | 外部寫入是否已完成？ |
| Interrupt Manifest | 指定等待、回覆 Schema、期限與執行關聯。 | 回覆消費後是否成功進入 Graph？ |
| Execution Manifest | 保存 Graph 設定與相關 Schema 版本。 | 新部署是否已具備資料遷移器？ |
| Task／Step | 記錄工作與步驟的持久狀態。 | 外部成功但本地補記失敗時，真實效果為何？ |
| Ledger／Result Reference | 查找執行嘗試、效果狀態與既有結果。 | 遺失的外部回應是否可由其他來源補回？ |

這份表格整理責任，沒有要求所有 Runtime 使用相同儲存結構。恢復時應先辨認哪份資料對哪個事實有權威，再處理矛盾，不能以最後寫入的時間取代語意判斷。

## Chat Gun 的恢復入口

`createConversationRecovery().recover()` 從 `interruptId` 尋找 Manifest，再核對 thread、Run、Task 與 scope。回覆已提供時，還會檢查 Manifest 是否 waiting，以及期限是否有效。

接著讀取 Checkpoint。找不到時回傳 `CHECKPOINT_NOT_FOUND` 的人工介入結果；讀到資料後才進入歷史整理、持久證據載入與執行位置分類。這條入口要求一份可找到的 Interrupt Manifest，不能由函式名稱推論它能接手所有沒有等待紀錄的程序故障。

概念順序如下，省略個別拒絕出口：

~~~text
尋找 Manifest 並核對關聯
  → 讀取 Checkpoint
  → 核對版本並整理歷史
  → 載入 Task／Step／Ledger 證據
  → 分類最後執行位置
  → 回傳續行、等待、回查或人工處理結果
~~~

原始碼可確認這些呼叫順序；`loadDurableEvidence` 與 Reconciler 的實際接線由 dependencies 提供，必須另外查證。

## 版本相容性先於歷史重用

Execution Manifest 保存 Graph ID、Graph 設定雜湊與 Schema 版本。`classifyExecutionManifestCompatibility()` 會比較 Manifest major version、Graph ID、設定雜湊及各 Schema 版本。

| 分類 | 查讀實作的判斷方向 | 後續責任 |
|---|---|---|
| compatible | 關鍵設定與 Schema 比較通過。 | 仍須檢查歷史及執行證據。 |
| migratable | Schema 有差異，但未觸及比較中的 major 差異。 | 需要具體遷移與驗證。 |
| incompatible | Graph、設定或受檢查的 major version 不符。 | 停止直接重用。 |

這個分類器沒有執行 migration。Sanitizer 對需要遷移的資料可回傳停放原因，不能把 `migratable` 顯示成已完成自動升級。部署版本也應依實際比較欄位理解，不能假設 Manifest 中每個欄位都已成為拒絕條件。

## 歷史整理保留什麼

恢復的訊息可能包含只有開頭的 Assistant Message、沒有對應結果的 Tool Call，或舊 executor 留下的重複結果。這些片段進入下一輪模型之前，需要先查明是否仍可配對。

Chat Gun `sanitizeRecoveryHistory()` 具有 valid、sanitized 與 parked 三種結果。它也保留未配對呼叫、未配對結果、重複結果、不完整參數、版本不相容與無法證明歷史等 reason code。

這些 code 代表不同處理分支，不能理解成所有異常都直接刪除後續行。Sanitized 結果保留整理後的 history 與原因；parked 結果交由人工處理，`recover()` 不繼續消費回覆。

歷史整理的目的在於產生可使用的對話與配對。刪除一份舊 tool result，不會撤銷它曾觸發的外部寫入，也不能把 Ledger 的 unknown 改成未提交。

## 最後執行位置的判斷順序

`classifyLastExecutionPoint()` 使用 Task、可選 Step、Ledger、Checkpoint 與 Manifest 證據。其判斷有順序：

1. Checkpoint 或指定 Task／Step 狀態符合終止條件，先分類為 terminal。
2. Ledger 為 committed 或 unknown，採用對應分類。
3. Manifest waiting 時分類為 waiting_user。
4. 符合尚未開始的組合時分類為 not_started；其他執行中證據可分類為 executing。
5. 仍無法歸類時保留 unknown。

Terminal 優先會阻止一般續行，後續效果查證仍可作為獨立工作保留。分類器能接受「Checkpoint 不存在」的組合；但目前 `recover()` 在呼叫它之前就會拒絕缺失的 Checkpoint，兩個層次的可達路徑應分開閱讀。

`executing` 只表示讀到執行中證據。實際接手者仍需核對 ownership、原 Worker 是否存活與工具的 replay 路徑，才能決定如何繼續。

## Resume 消費之後的窗口

在 waiting_user 分支，Recovery 驗證 response Schema、以已保存事件的最大序號初始化 allocator，再嘗試消費指定等待。Confirmation 另透過提供的 authorizer 消費；缺少它時不直接批准。

消費成功後，Checkpoint Adapter 依序執行：

~~~ts
await graph.updateState(config, { messages: sanitizedHistory });
return graph.invoke(new Command({ resume: response }), config);
~~~

這是查讀 Adapter 的核心順序。Manifest 消費、歷史更新與 Graph invoke 分屬不同呼叫，程序可在任兩步之間停止。回覆只能消費一次，也仍可能遇到「已消費但尚未接手」的空窗，需要恢復接手紀錄或其他可證明的補接機制。

Adapter 沒有決定底層 Checkpointer 的持久性。Deep Researcher 查讀的位置使用 `MemorySaver`，單憑它無法保證程序重啟後仍可找回狀態；部署採用哪種 Checkpointer，須由實際 Graph 與環境設定確認。

## CCB 的 Session 恢復對照

CCB `QueryEngine.ts` 依 `isSessionPersistenceDisabled()` 決定是否保存 transcript。查讀區段有等待 `recordTranscript()` 完成的路徑，也有 `void recordTranscript()` 的非同步寫入；consumer 是否等到寫入完成會影響故障時可讀回的範圍。

`sessionStorage.ts` 提供 transcript 記錄與載入 helper；`sessionRestore.ts` 則整理恢復後的 messages、file history、agent 設定與其他 session 狀態。這些是 Session 恢復的具體接點，無法單獨證明每份工具外部效果都有 durable Ledger 或恢復交易。

對照兩個來源時，可以問它們如何恢復可供推理的歷史，再分別追蹤工具效果。Session messages、Graph Checkpoint 與外部業務紀錄各有保存契約。

### Transcript 如何找回一條對話

CCB `loadTranscriptFromFile()` 在 JSONL 路徑先呼叫 `loadTranscriptFile()`，找出 leaf message，再由 leaf 向 root 建立 conversation chain。沒有訊息或沒有有效鏈時會拋錯，並非讀入檔案中的所有列就直接接續。

這說明恢復還有結構條件：訊息存在、父子關聯可追溯，以及選定哪個分支是三件事。`recordTranscript()` 也會先辨認已記錄訊息，保留前綴中的 chain participant，再插入新訊息鏈。部分資料仍在檔案裡，不代表它一定屬於目前恢復的可達分支。

因此驗證 Session 恢復時，除了查看訊息數量，也要核對 leaf、父子鏈、工具配對及 compact 後的有效歷史。這些結構恢復仍沒有替外部操作建立未提交證據。

## 如何驗證恢復資料

- 分別讓 Manifest、Checkpoint 與 Ledger 缺失，核對每種組合的出口及原因。
- 注入部分訊息、重複結果與不相容版本，確認 parked 時沒有消費回覆。
- 在 Resume 消費後、歷史更新後與 Graph invoke 前停止程序，核對重新接手方式。
- 對同一部署重啟程序，確認 Checkpointer 實際可讀回；再測不同 Worker 是否讀到同一份狀態。

上述是驗證設計。本輪讀取 Recovery 實作及既有測試安排，未執行程序重啟、資料庫或跨 Worker 演練。

## 本篇的工程判斷

我會先列出恢復需要的事實及其權威來源，再決定讀取順序。資料彼此矛盾時，保留原因與未解效果，能讓後續回查知道缺少什麼；直接重跑節點會讓這些矛盾變成第二次操作。

## 實作參照與適用範圍

- Chat Gun：`runtime/recovery/conversation-recovery.ts`、`langgraph-checkpoint-adapter.ts`、`last-execution-point.ts`、`execution-manifest.ts` 與 `recovery-sanitizer.ts`，路徑相對 `backend/src/`。
- CCB：`src/QueryEngine.ts`、`src/utils/sessionStorage.ts` 與 `src/utils/sessionRestore.ts`。
- [來源與檔案指紋](./README-zh-TW.md#來源與查證狀態)記錄本地查讀範圍；本文沒有宣稱通用 Recovery 入口已在所有正式 Graph 啟用。

## 接續閱讀

下一篇[冪等與重放識別](./02-idempotency-and-replay-identity-zh-TW.md)追查恢復之後如何辨認同一次呼叫、下一個 Attempt 與相同業務效果。
