# 冪等與重放識別

[English](./02-idempotency-and-replay-identity.md) | [繁體中文](./02-idempotency-and-replay-identity-zh-TW.md)

> 以下內容所提到的
> CCB 為 [claude-code-best](https://github.com/claude-code-best/claude-code) 的簡稱。
> Chat Gun 為 [chat-gun](https://github.com/HsienW/chat-gun)。

[回到導覽](./README-zh-TW.md) · 前一篇：[Checkpoint 與恢復資料](./01-checkpoints-and-recovery-state-zh-TW.md) · 下一篇：[效果帳本與提交邊界](./03-effect-ledger-and-commit-boundaries-zh-TW.md)

> 使用者送出一次操作，連線中斷後又重送。Runtime 建立新 Run，模型也產生新的 Tool Call ID；外部系統看見的業務要求卻相同。若只比較 Run ID，系統可能再次執行；若只比較工具名稱，又會把兩次合法的不同操作合併。

**去重需要先定義「相同」的範圍。** 請求、邏輯呼叫、實體 Attempt 與業務效果有不同身分，重用條件也隨之改變。

## 五種身分各自回答什麼

| 身分 | 回答的問題 | 不能取代的判斷 |
|---|---|---|
| Request Dedup Key | 同一身分是否重送同一路由的請求？ | 工具是否產生相同業務效果？ |
| Replay Key | 是否重放同一 Run 中的指定邏輯呼叫？ | 新 Run 是否重做同一業務操作？ |
| Tool Execution ID | 哪份持久工具執行紀錄？ | 實際派送了幾次？ |
| Attempt ID | 哪一次實體嘗試？ | 多次嘗試是否應合併成同一效果？ |
| Business Effect Key | 哪個可信 scope 中的業務效果？ | 外部服務是否已接受同一冪等鍵？ |

這些識別碼需要可查回的關聯。讓所有欄位都使用同一個隨機值，會失去辨別邏輯呼叫與實體重試的能力；只增加更多 ID，也無法自動建立去重契約。

## Chat Gun 的鍵如何組成

本輪補查安裝的 `@gun-ai/harness-kernel`，版本為 `0.1.0-alpha.1`。`identity.ts` 提供雜湊與 UUID dependencies，kernel 保存組成規則；查讀 `src/side-effect.ts` 與套件 export 使用的 `dist/side-effect.js`，可確認以下欄位。

Replay Key 的輸入順序為：

~~~text
runId
stepId
logicalToolCallId（缺少時使用 toolCallId）
callIndex
toolName
toolVersion
~~~

Kernel 驗證必要字串與非負安全整數 `callIndex`，再將這些值交給 wrapper 的 hash。Chat Gun wrapper 對陣列的 JSON 表示計算 SHA256。

Request Dedup Key 使用 tenant、principal、route namespace 與 client key。Attempt identity 則保留 Tool Execution ID、正整數 `executionAttempt`，另產生 UUID。這些 helper 提供識別規則，入口是否採用及如何驗證 client key，仍由實際 consumer 決定。

## 同一重放鍵與不同參數

Chat Gun runner 讀到既有 Replay Key 後，先比較 `requestHash`。相同 key 配上不同 hash，回傳 `SIDE_EFFECT_REPLAY_CONFLICT`；既有紀錄未 committed 時，回傳需要回查，而非再次呼叫工具。

這個分支保護的是已記錄的邏輯呼叫。`requestHash` 如何涵蓋 canonical input、哪些欄位應穩定化，仍需對回建立它的 Pipeline 與工具契約。若上游把不同語意的要求算成相同 hash，runner 無法從 hash 值還原遺失的差異。

同樣地，模型產生新 ID 只表示另一份提案；系統仍應核對它是否指向同一業務效果。模型重新敘述要求，不能清除先前 unknown 的效果紀錄。

## 業務效果的去重範圍

`deriveBusinessEffectKey(input, scope)` 由 descriptor 定義，再經 wrapper 雜湊。資料庫唯一約束使用：

~~~text
tenant_id + scope_id + business_effect_key
~~~

因此新 Run 雖然改變 Replay Key，仍可能由相同業務鍵關聯到既有效果。業務鍵應依操作語意設計，例如可信訂單與指定退款意圖；這是教學例子，沒有宣稱來源庫提供退款工具。

金額、資源版本、對象與重複操作是否允許，都會影響鍵的設計。鍵太粗會阻止合法的新操作，鍵太細則可能讓重送變成新效果。Scope 也必須來自可信 Context，不能直接採用模型填入的 tenant 名稱。

資料庫另外約束外部 system namespace 與 operation ID 的配對及唯一性。這能維護本地已取得的外部關聯；外部 API 是否接受並持久去重同一操作，仍需要服務端契約。

### Production MCP 的業務鍵範例

`production-runtime-tool-descriptors.ts` 對查讀的非唯讀 MCP 工具，以 server name、tool name、tool version、tenant、scope 與 input 建立 canonical JSON，作為業務鍵的原始值。Canonicalizer 遞迴排序物件鍵，保留陣列元素順序。

這個規則可以讓物件欄位排列不同的相同輸入取得穩定表示；它也保留了明確限制：工具版本或輸入值改變，鍵就可能改變。兩份業務上相同的要求若使用不同預設值、不同時間字串或不同表示法，仍需工具自行定義語意正規化；反過來，業務允許再次執行完全相同參數時，也不能只靠參數相同判斷為重送。

Pipeline 建立 runner identity 的查讀區段則使用 canonical Tool Call ID，並將 `callIndex` 設為 `0`。Kernel 支援 call index，不代表這個 consumer 已為同名呼叫分配遞增索引；此入口還依賴 Tool Call ID 的正確建立與配對。

## TTL 到期之後

`PgIdempotencyGuard` 保存 namespace、resource key、version 與 locked／completed／failed 狀態。Kernel 將鍵序列化為 `namespace:resourceKey:vversion`，這套紀錄與 runner 的 Replay Key／效果 Ledger 分開使用。

Guard 可刪除 failed 紀錄後重新取得，也可在過期條件成立時嘗試回收。TTL 只限制本地紀錄有效期；遠端操作可能仍在執行，或已提交而本地未補記。

在查讀方法中，完成與失敗更新依序列化 key 定位，沒有在該更新條件中核對 owner token。若應用允許 TTL 回收後由另一執行者接手，就要檢查舊持有者晚到的更新如何受控。這是需要消費端與並行情境驗證的窗口，不能只因存在 Guard 便宣稱具備跨 Worker fencing。

## 已提交結果何時可以重用

Ledger committed 讓 Runtime 知道已有提交紀錄，模型需要的內容則可能放在 Result Reference。`PgResultReferenceStore.resolve()` 會檢查 scope 相容、tool version、cache state 與 `policy.isReusable()`，通過後才呼叫 `resolveResultRef()` 取得 payload。

| 情況 | 查讀路徑的處置方向 | 應保留的事實 |
|---|---|---|
| 相同重放鍵、不同 request hash | 回報 conflict。 | 原操作與新參數衝突。 |
| 既有執行尚未 committed | 回報需要回查。 | 尚無可直接重用的 committed 結果。 |
| Scope／版本不符或結果不可重用 | 拒絕結果重用。 | 既有效果仍可能已提交。 |
| Payload 缺失 | 無法交付既有結果。 | 內容遺失不等於效果不存在。 |
| 結果符合重用條件 | 返回既有結果。 | 結果的來源為 cache 或已回查資料。 |

結果重用是資料存取與相容性判斷，也仍受產品授權要求約束。不能把 cache 可讀直接擴張成新操作的執行授權。

## CCB 的 ID 對照

CCB 的 tool-use／tool-result ID 支援訊息配對，StreamingToolExecutor 也以工具關聯收集結果。它們適合辨認結果屬於哪份提案；本輪沒有從這些 ID 找到與 Chat Gun 相同的跨 Run 業務效果唯一約束。

若要替 CCB 中某個工具建立外部冪等，仍需查該工具的 adapter、外部 API 與持久紀錄。Transcript 中出現一次成功訊息，只是可用證據的一部分。

## 如何驗證重放識別

- 同一 Replay Key 改變參數 hash，確認回傳衝突且不派送。
- 新 Run 使用相同業務鍵，核對既有 committed 效果與結果重用條件。
- 讓 payload 不可取得、scope 改變或版本不符，確認沒有以內容缺失觸發重做。
- 在 TTL 回收後交錯新舊持有者的完成更新，核對 owner 與外部效果紀錄。
- 對外部冪等鍵測試保存期限及同鍵不同參數的服務端行為。

本輪讀取識別、Guard、Ledger 與 Result Reference 實作，未執行外部 API 或多 Worker 重送測試。

## 本篇的工程判斷

我會先寫清楚業務上允許重複的操作，再設計鍵。重試應增加 Attempt 證據，重放應查回原邏輯呼叫，新業務操作則需要自己的語意身分；這些關聯能讓故障後的處理保有依據。

## 實作參照與適用範圍

- Chat Gun：`runtime/side-effect/identity.ts`、`tool-execution-runner.ts`、`business-effect-ledger.ts`、`result-reference-store.ts` 與 `runtime/idempotency/idempotency-guard.ts`，路徑相對 `backend/src/`。
- Chat Gun production MCP：`backend/src/tools/production-runtime-tool-descriptors.ts` 的 canonical input 與業務鍵；`backend/src/runtime/tool-dispatch/pipeline.ts` 的 identity 建立。
- Kernel：`@gun-ai/harness-kernel` 的 `src/side-effect.ts`、`dist/side-effect.js` 與 `src/idempotency.ts`。
- [來源與檔案指紋](./README-zh-TW.md#來源與查證狀態)記錄套件及本地檔案；本文沒有建立對所有外部服務通用的 exactly-once 保證。

## 接續閱讀

下一篇[效果帳本與提交邊界](./03-effect-ledger-and-commit-boundaries-zh-TW.md)沿實際派送順序，追查外部效果與本地提交之間的故障窗口。
