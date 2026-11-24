# 多程序環境中的執行身分與狀態權威

本文件保存第一版的 Munder Difflin 個案素材與待實測項目。六篇正文採用「共同執行責任 → 三種業務差異」的比較架構，閱讀請從 [Runtime Foundations 總覽](./README-zh-TW.md) 開始。

同一個 Worker 在原來的終端視窗重新啟動，舊程序此時才送出退出通知。畫面該顯示誰的狀態？這是 Munder Difflin 在 PTY 管理程式中實際處理的競態，也適合貫穿本組文章。Chat Gun 提供應用端的 Run 與工具執行案例，Claude Code Best（CCB）提供 CLI 內部迴圈的程式對照，Munder Difflin 則讓讀者看到，當產品從另一個程序管理 CLI 時，手上有哪些證據。[PTY 回呼的歸屬檢查][pty]

本稿以 Munder Difflin 的 Claude Code Worker 為例。原始碼與設計文件固定在 [96ae43e][rev]。文中「原始碼」指固定版本可查讀的分支，「設計」指協定預期行為，「待實測」列出取得執行證據的方法。第 06 篇的檔名與任務皆為教學情境。

## 01　外層能控制一次執行的哪些部分？

**落點**　在[實際落地的 Agent Runtime 是什麼？](./01-what-is-agent-runtime-zh-TW.md)的 CCB 與 Chat Gun 實作對照後，加入「從 Electron 看一個 CLI Worker」。沿用前文的讀檔例子，從程序外再看一次。

**試寫**　Worker 要讀取工作區裡的一份檔案。Munder Difflin 的主程序先以指定的工作目錄、命令列參數與環境啟動 CLI，並從虛擬終端（PTY）接收輸出。接下來，選哪個工具、如何把讀取結果送回模型、是否再推理一輪，都在 CLI 內發生。若該 CLI 有回報工具事件，外層便可把事件連回 Worker。單看終端有新文字，只能確定這次 PTY 收到了輸出。[架構文件][arch]、[程序建立與退出路徑][pty]

**成稿配例**　以「主程序已啟動 CLI」「CLI 已讀到協作指令」「Worker 已能回信」作為三個可獨立查驗的敘述。Windows 曾因多行啟動參數遭截斷，出現第一項成功、後兩項失效的情況。可把這筆歷史故障放在段末，說明為什麼驗收要走到回信。[Windows 修復紀錄][changes]

## 02　同一個 Worker 重啟，哪些 ID 還有效？

**落點**　在 [Execution Identity 與狀態權威](./02-execution-identity-and-state-authority-zh-TW.md) 的 Identity Chain 後加入下表，再接回原文的 generation 競態。表中五種身分各自標示一個物件，一次委派可以跨越多個 CLI 回合。

| 身分 | 用哪份資料認出它 | 同一 Worker 重啟後如何判讀 |
|---|---|---|
| Agent ID | Hive 登錄中的角色與工作目錄 | 恢復原登錄時可沿用。先核對所屬 Hive 及登錄項。 |
| PTY ID | 主程序以 `opts.id` 查目前的 PTY Session 物件 | 重新啟動可重用同一個鍵。要認出活躍程序，還要比對物件歸屬。 |
| CLI Session ID | Provider 的對話識別碼，Hive 保存目前值與歷史候選 | 實際續接取決於 Provider 是否接受該識別碼及對應紀錄是否存在。新對話要另記。 |
| Task ID | 任務帳本內的工作項目、負責人與狀態 | 回到原任務紀錄即可繼續對帳。重啟本身無法判定該工作是否完成。 |
| Message ID | 信件本身的 `id`，回信另用 `in_reply_to` 指回原件 | 重讀同一封持久化信件仍可用原 ID 核對。要確認內容已執行，另查接收端及工作結果。 |

**試寫**　舊程序 A 和替代程序 B 可以先後使用同一個 PTY ID。A 遲來的退出回呼拿著自己的 Session 物件查表，看見目前掛在這個 ID 下的是 B，就直接返回。這個比較保住 B 的輸入通道，也避免 UI 收到一次歸錯對象的退出通知。它保護的是這個主程序內的回呼，跨應用程式重啟的對話延續另由 Hive 與 CLI 處理。[PTY 原始碼][pty]、[Hive 的 Session 保存路徑][session]、[任務與訊息設計][hive]

**待實測**　分別記錄「重啟 Worker 並續接」「以原角色開新對話」「關閉再啟動整個應用程式」三組前後值。讓同一筆 Task 留在帳本，再檢查其負責人、回覆關聯與對話識別是否各自延續。

## 03　一個回合結束後，誰決定還有工作？

**落點**　在 [Run Loop 與狀態轉移](./03-run-loop-and-state-transitions-zh-TW.md) 的 CCB query loop 旁加一個收件案例。先沿著 CLI 走到回合終點，再切換到外層 Router 的視角。

**試寫**　CLI 內層收到工具結果後，可以再次呼叫模型，也可以收束這一輪。外層 Router 仍有自己的工作，例如把另一個 Agent 的請求從 outbox 投到 Worker 的 inbox。Hive 的 Claude 路徑設計會在 Stop hook 檢查待收訊息，必要時要求繼續工作。信箱已空時，這條路徑便沒有延續該回合的理由，Worker 仍可保留終端等下一件工作。協定也區分需要回答的請求與可收束的完成通知，並設計往返次數上限，以免兩個 Worker 一直互答。[Hive 控制流程與訊息協定][hive]

**待實測**　分別提供空信箱、一封需要回覆的請求、一封完成通知。每次記錄 Router 是否投遞、hook 收到何種回應、CLI 是否繼續，以及回合結束後程序是否仍存活。據此在圖上分別標記回合結束、訊息往返結束與程序退出。

## 04　重啟之後，可以從哪裡接下去？

**落點**　在[持久化、Interrupt 與 Resume](./04-persistence-interrupt-and-resume-zh-TW.md)的「Transcript、Memory 與 Checkpoint」後加入一個重啟情境。

**試寫**　重啟一位正在整理文件的 Worker，第一個問題是它接回了什麼。Hive 的 `recordSession` 先確認角色存在、Session ID 符合 Provider 格式，再保存新值及歷史候選。歷史修復紀錄另記載，Claude 的恢復路徑曾因缺少 Session ID 而失效，後來新增了取得 ID 的來源。查到 ID、找到 Transcript、CLI 成功接回原對話，分別是三個驗證點。[Hive 原始碼][session]、[修復紀錄][changes]

| 恢復材料 | 重新開工時能回答的問題 | 還需要的證據 |
|---|---|---|
| 角色記憶 `memory.md` | 這位 Worker 留下哪些下次可讀的知識？ | 本次啟動是否實際載入相關內容 |
| Transcript | 先前對話記錄到哪一句或哪次工具呼叫？ | 紀錄是否屬於目標對話，最後動作是否已有結果 |
| CLI Session Resume | CLI 能否用候選 ID 與可用的 Transcript 重新開啟該對話？ | 啟動後核對對話歸屬及可讀歷史 |
| Task 紀錄 | 誰負責這件工作，最後登記的進度及結果參照是什麼？ | 結果是否存在，先前操作是否需要回查 |

**待實測**　用同一個可辨識的對話分別測試 Transcript 存在與缺失。若重啟前曾等待使用者批准，另查等待識別、核准範圍及其消費紀錄，再判斷是否能繼續該步驟。

## 05　哪些證據足以改變狀態？

**落點**　在 [Agent Runtime 設計原則](./05-runtime-design-principles-zh-TW.md) 的「每種狀態只有一個權威來源」後補一張判讀表，於故障注入段落回引。

| 要判定的事 | 優先查的證據 | 證據有缺口時 |
|---|---|---|
| 程序仍存活 | 目前 PTY Session 所屬程序的存活檢查及退出事件 | 記錄最後確認時間，重新核對程序身分。 |
| CLI 已閒置，可接受輸入 | CLI 活動訊號、佇列狀態、終端是否留有使用者草稿 | 保留待送訊息，核對阻擋原因及訊號時效。 |
| 任務已完成 | 對回 Task 的完成回報及可檢查的交付物 | 保留「已回報」或「待核對」的判讀，再查交付物。 |

**試寫**　畫面上 Worker 安靜了，可能是工具尚未回來，也可能是使用者在終端打了一半的字。Munder Difflin 的佇列文件把 CLI 閒置與是否適合自動送字列為不同條件。值班者若漏接一次 hook，可以先核對 PTY 是否還屬於原程序，再查信箱、對話及任務紀錄。這幾份資料回答不同問題，應在畫面或事件紀錄裡分開交代。[終端輸入契約][queue]、[PTY 原始碼][pty]

**待實測**　比較「刻意缺少一筆 hook」「操作人員留下草稿」「回報完成但找不到產物」三種情況，記下介面顯示、可用證據及人員能確認的最終狀態。

## 06　一次 Worker 委派的完整時序

**落點**　在[一次 Run 的完整走法](./06-one-run-walkthrough-zh-TW.md)的 CCB 對照路徑後新增桌面 Worker 案例。沿用原篇逐步查證的寫法。以下以「找出 README 中失效的相對連結，回報檔名與行號」為例，讓讀者能自行檢查結果。

| 時點 | 執行者與動作 | 應留下的證據 |
|---|---|---|
| 1 | Orchestrator 建立工作項目，指定 Worker 與要檢查的檔案 | Task ID、Agent ID、工作目錄及驗收條件 |
| 2 | 主程序準備角色與 Hive 協作指令，將啟動參數與環境交給 PTY，啟動 CLI | 參數中協作指令的完整性、PTY ID、程序身分及工作目錄 |
| 3 | Orchestrator 寄出交辦訊息，Router 投遞，Worker 讀取 | 原 Message ID、收件人、收件或讀取證據 |
| 4 | CLI 在自己的回合讀取 README 並檢查連結 | CLI Session ID、可用的工具事件或 Transcript、檢查結果 |
| 5 | Worker 回信，接收端對回交辦，核對檔名及行號，依結果更新任務 | 回信 ID、原訊息關聯、結果與 Task 最終紀錄 |
| 6 | 此次演練在工作驗收後關閉 Worker，由主程序處理退出 | 目前 Session 的退出通知、exit code、signal 與清理結果 |

**試寫**　同一封回信可以在 Worker 退出前送達，也可能在退出後才由另一端讀取。驗收時先用 Message ID 把請求與回信連起來，打開指定的 README 核對行號，再回到 Task 紀錄。最後才看 PTY 的退出事件，確認關閉的是哪一個程序。這份時序分別交代了工作結果與程序的結局。[Hive 路由設計][hive]、[程序退出處理][pty]

**待實測**　在第 3 步投遞後重啟 Worker，保存重啟前後的對話識別、收件紀錄與回信。這一輪要回答兩件事：Worker 接回哪段對話，以及同一件工作有沒有被執行兩次。等到一筆實際委派能串起這六步，再將教學時序補成實測案例。

[rev]: https://github.com/HarnessMD/munder-difflin/commit/96ae43e3292447f68af6904afb353f6f5bdaa412
[pty]: https://github.com/HarnessMD/munder-difflin/blob/96ae43e3292447f68af6904afb353f6f5bdaa412/src/main/pty.ts#L895-L975
[session]: https://github.com/HarnessMD/munder-difflin/blob/96ae43e3292447f68af6904afb353f6f5bdaa412/src/main/hive.ts#L2209-L2245
[arch]: https://github.com/HarnessMD/munder-difflin/blob/96ae43e3292447f68af6904afb353f6f5bdaa412/docs/ARCHITECTURE.md
[hive]: https://github.com/HarnessMD/munder-difflin/blob/96ae43e3292447f68af6904afb353f6f5bdaa412/HIVE.md
[queue]: https://github.com/HarnessMD/munder-difflin/blob/96ae43e3292447f68af6904afb353f6f5bdaa412/docs/message-queue.md
[changes]: https://github.com/HarnessMD/munder-difflin/blob/96ae43e3292447f68af6904afb353f6f5bdaa412/CHANGELOG.md
