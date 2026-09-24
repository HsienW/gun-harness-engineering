# Agent Harness

本區整理 Agent 的行為控制：使用者輸入如何進入系統、模型能看見哪些內容、可使用哪些能力、行動前如何取得授權，以及使用者如何在執行中介入。

## 閱讀順序

1. [Foundations](./foundations/README.md)：先建立定義、控制面與 Harness／Runtime 邊界。
2. Input and Context：輸入正規化、上下文組裝與信任邊界（規劃中）。
3. Behavior and Capabilities：行為規則與能力配置（規劃中）。
4. Action Governance：工具風險、授權與人工確認（規劃中）。
5. Interaction and Feedback：執行中互動與行為評估（規劃中）。

## 撰寫約定

每篇依序說明作者判斷、真實場景、程式落點與適用邊界。引用案例時區分「Claude Code 舊版程式已觀察」、「Chat Gun 正式路徑已接入」、「模組存在但未接入」與「規劃中」。尚未核對的能力不寫成已完成。
