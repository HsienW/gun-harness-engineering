# Agent Harness 基礎篇

[English](./README.md) | [繁體中文](./README-zh-TW.md)



這五篇先定義 Harness 負責什麼，再追問決策如何進入執行，最後用一次檔案修改任務核對整條路徑。依序閱讀，後一篇會把前一篇留下的問題推進到具體流程。



## 閱讀順序

1. [實際落地的 Agent Harness 是什麼](./01-what-is-agent-harness-zh-TW.md)：界定 Harness 在 Agent Product 與 Runtime 之間的責任；後續篇章沿用這裡的名詞。
2. [Harness 的控制層](./02-control-surfaces-zh-TW.md)：把定義拆成輸入、Context、行為與能力、行動治理、互動與回饋五個可檢查的決策面。
3. [Harness 與 Runtime 的邊界](./03-harness-runtime-boundary-zh-TW.md)：追蹤 Harness 交付哪些執行條件、Runtime 何時回來請求決策，以及狀態與失敗由誰負責。
4. [Agent Harness 設計原則](./04-design-principles-zh-TW.md)：從前述邊界整理工程取捨，包括依任務選取 Context、區分指令與資料權威、檢查行動，以及從正式入口驗證規則。
5. [一次任務的完整走法](./05-one-turn-walkthrough-zh-TW.md)：用檔案修改情境串起輸入、Context、工具結果、授權、失敗與完成證據，檢查前四篇的責任分工如何運作。
6. 接著可讀 [Input and Context](../input-and-context/README-zh-TW.md)，進一步展開輸入正規化、模型可見材料、來源權威與 Memory。



* 文中的實作參照庫，應分清已觀察的程式路徑、已接入入口的能力、僅有模組與設計提案；教學案例也不能當成正式環境的執行紀錄。

  * \[claude-code-best](https://github.com/claude-code-best/claude-code)
  * \[chat-gun](https://github.com/HsienW/chat-gun) 



