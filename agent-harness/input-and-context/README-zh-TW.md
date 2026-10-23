# Input and Context

[English](./README.md) | [繁體中文](./README-zh-TW.md)

本區接在[基礎篇](../foundations/README-zh-TW.md)之後，追蹤 Harness 如何接收任務、決定模型能看見的材料，並保留指令與資料的界線。五篇請依序閱讀；各篇區分已讀到的程式路徑、已接入正式入口的能力與作者的設計判斷，尚需實際執行證據的部分，也在文中標明。

## 閱讀順序

| 文章 | 要回答的問題與篇章邊界 | 主要參照 |
|---|---|---|
| [01. 輸入正規化](./01-input-normalization-zh-TW.md) | 模型執行前，哪些內容會成為任務輸入？寫到正規化結果與來源為止。 | claude-code-best 的 `processUserInput`；Chat Gun 的 `normalized-agent-input.ts`、`read-normalized-agent-input.ts`、`execution-composition-root.ts`。 |
| [02. Context 組裝](./02-context-assembly-zh-TW.md) | 哪些區塊進入模型呼叫，順序與容量如何決定？ | claude-code-best 的 query 準備流程；Chat Gun 的 `context-integration.ts`、`context-source.ts`、`agent-context-assembler.ts`。 |
| [03. Context 信任邊界](./03-context-trust-boundary-zh-TW.md) | 哪些來源可下指令，哪些只能當任務資料？ | Chat Gun 的來源分類與當前任務優先序；基礎篇整理的指令與工具輸出邊界。 |
| [04. Memory 與壓縮](./04-memory-and-compaction-zh-TW.md) | 記憶提取與壓縮如何延續任務，又不蓋過本次要求？ | Chat Gun 的 Memory recall、容量、壓縮與 manifest；claude-code-best 各自獨立的 compacting 路徑。 |
| [05. 從輸入到 Context](./05-case-input-to-context-zh-TW.md) | 能否沿著一次請求，核對模型實際看見的內容？ | Chat Gun chatbot 入口，逐項標出納入、排除與失敗位置。 |

各篇依目前可讀的程式路徑寫成。第四篇若要宣稱某部署的端到端 Memory 行為，仍須核對 provider 設定與相關 Agent 入口；第五篇的教學情境則需要一筆可重現的請求及其 Context manifest，才能改寫成實際執行紀錄。容量優先序只能說明取捨，不能單憑它宣稱已擋下 Prompt Injection。

## 證據寫法

實作主張要附上程式庫與路徑。「已觀察」表示讀過程式碼；「已接入」須找到正式入口的呼叫；「設計判斷」則是作者提出的規則。引用舊對話或基礎篇時，仍須回到 Chat Gun 當前分支核對現況。先前的 Codex 對話可補充 Chat Gun 程式庫脈絡；claude-code-best 的路徑索引沿用基礎篇，並在各篇正文中逐一查證。
