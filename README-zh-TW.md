# Gun Harness Engineering

[English](./README.md) | [繁體中文](./README-zh-TW.md)

<p align="center">
  <img src="./assets/00-gun-harness-engineering-logo.webp" alt="chat-gun"/>
</p>

這是一套討論 AI Agent 在真實產品中受到約束、完成執行、故障恢復的知識庫

- 這些內容起源自個人對於 [chat-gun](https://github.com/HsienW/chat-gun) 的實戰加上對 [claude-code-best/claude-code](https://github.com/claude-code-best/claude-code) 的研究與思考。
- 本庫不會從 Agent Infra 的角度出發闡述介紹各種 Harness 論文，而是從產品實踐的視角看 Harness 的定位與能力，因為這才是業務落地的核心。

## 首先！不用技術名詞，你能說清楚 Harness 跟 Runtime 是什麼嗎?

> 如果你回答不出來或答案有模糊點，這個庫可以幫助你更清晰的區分兩者

### 讓我來回答: 我會用比賽來舉例，例如: 一場鐵人三項的比賽

#### Part1: Harness 跟 Runtime

<p>
  <img src="./assets/01-harness_runtime.webp" alt="source: hsien-wei" width="768" />
</p>

- **Harness** = 主辦單位負責規劃賽事大方向的角色
  - 例如: 舉辦日期 / 比賽規則 / 籌借場地 / 資金 / 找贊助 等等有方向性且趨於固定的任務

- **Runtime** = 設備組 / 補給組 / 醫療組 等等是在比賽進行時，真正動態工作、靈活調整的角色
  - 例如: 比賽進行時，依照天氣會調整補給點的多寡和位置 / 醫療組的站點 / 設備組的帳篷 等等這些都是進行時，靈活增刪跟機動的

- **Context** = 比賽進行中不斷更新的內容，因為內容隨著時間一直再更新與增加，所以計分表 / 媒體報導 / 賽況轉播 只會抓重點來說明，這就跟上下文中一旦太長就需要壓縮摘要跟記憶相同道理
  - 例如: 多少人參賽 / 目前進行到哪個項目 / 當下誰領先 / 是否有破紀錄 等等

#### Part2: 其餘 Agent 能力

<p>
  <img src="./assets/02-harness_runtime.webp" alt="source: hsien-wei" width="768" />
</p>

- **LLM** = 賽事主辦單位中，擔任本次比賽統籌的團隊，負責規劃構想，找出需要哪些協力外部單位的角色
  - 例如: 比賽設備是否要找額外協力單位 / 場地跟誰租借 / 設備怎麼獲取 / 是否贊助廣告商支持

- **MCP** =  主辦單位中統一對外的溝通窗口，負責按照統籌團隊給出的所需清單聯繫外部的協力單位
  - 例如: 聯繫場地所有者 / 訂好租借日期 / 通知採購補給品廠商 / 聯繫提供醫療設備廠商

- **Tool** = 比賽清單上寫的每個外部協力單位
  - 例如: 補給品廠商 / 贊助商 / 採訪媒體

- **Log / Trace** = 比賽舉辦過程的紀錄，用來檢討比賽爭議或改進下一次比賽舉行的材料
  - 例如: 裁判 / 攝影師 / 計時器

#### Part3: 人類負責的範圍

<p>
  <img src="./assets/03-harness_runtime.webp" alt="source: hsien-wei" width="768" />
</p>

- **Human** = 主辦單位的老闆，透過 Log / Trace 來觀測跟控制以下角色，讓下次比賽舉辦的更好。
  - 統籌團隊(LLM) + 主辦單位 (Harness) + 設備組/補給組/醫療組 (Runtime) + 比賽過程內容 (Context)
  - 協力單位(Tool)
  - 替換新的統籌團隊(Change Model)
  - 執行別種類型的比賽(Change Business)

> 所以用白話來說可以理解為 Runtime 是在 Harness 的規劃好的範圍內運行 + 靈活調整的角色，而 Context 是內容，Tool 是額外能力
> 而人類使用 Harness + Runtime 來控制 LLM 和 Agent 滿足不同的 Business 的操作

## 用技術名詞來說，我所認知的 Harness 與 Runtime

- **Agent** = Harness + LLM 這是目前業界公認說法，在這基礎上往下拆分，如下圖:

<p>
  <img src="./assets/01-gun-harness-pyramid.webp" alt="gun-harness-pyramid" width="768" />
</p>

圖中由下而上可以看到四個部分 (最上層綁定的領域是跟隨業務來替換的，越往下越是 Agent 的通用能力)

1. Infra: 這是屬於模型範圍，例如: GPT-5 / GPT-6 和 Claude Opus / Sonnet 等系列都是大眾熟悉的模型。
2. Harness: 是模型的控制層，決定能力跟大方向的控制器，例如: 構建與編排層（Prompt / Workflow）、連接層（API / MCP 等協議、能力層（Skills / Tools） 都在這層。
3. Runtime: 是大方向已經被控制器定好之後，真正運行時的管控，例如: 沙箱環境、狀態與記憶、重試機制、節流、權限治理、Trace 觀測都在這。
4. Business: 是具體業務在前三層之上疊加不同 Domain 落地的範圍，其實也是現在 FDE 在做的事，例如: 教育、金融、航空等等不同業務領域的 Agent 落地。

**因為 Business 強綁定業務、而 Infra 歸屬模型能力跟模型訓練範疇，所以本庫只會聚焦在以下兩個的主線:**

- **Harness**： 產品如何決定 Agent 看見什麼、可以使用哪些能力、行動如何取得授權，以及人如何介入、跟追蹤路徑。
- **Runtime**： 一次 Agent Run 如何推進狀態、執行模型與工具、處理外部副作用、從失敗恢復，並留下可供營運者檢查的證據。

## 為什麼建立這個知識庫

Agent 架構常被寫成一張元件清單：Prompt、Memory、Tool、Workflow、Tracing、Evaluation。這類清單沒有回答誰擁有決策權、執行在哪裡跨越信任邊界，也沒有說明工具其實已成功、回應卻遺失時該怎麼辦。

本庫從幾個更接近實際運行的問題出發：

- 哪些責任屬於 Harness，哪些屬於 Runtime？
- 使用者輸入、歷史記憶與外部資料衝突時，誰有較高權威？
- 誰可以批准一次會改變外部世界的行動？
- 工具失敗如何回灌模型，又不破壞對話結構？
- 哪組識別碼可以串起使用者請求、Run、模型請求、工具呼叫與副作用？
- 程序崩潰後，哪些狀態可以安全恢復？
- 哪些能力已經走過正式執行路徑，哪些仍停在模組或規劃？

## 知識庫地圖

~~~text
gun-harness-engineering/
├─ README.md
├─ README-zh-TW.md
├─ agent-harness/
│  ├─ foundations/
│  ├─ input-and-context/          # 規劃中
│  ├─ behavior-and-capabilities/  # 規劃中
│  ├─ action-governance/          # 規劃中
│  └─ interaction-and-feedback/   # 規劃中
└─ agent-runtime/                 # 規劃中
   ├─ foundations/
   ├─ execution-lifecycle/
   ├─ model-and-tool-execution/
   ├─ effects-and-recovery/
   └─ events-and-operations/
~~~

### Agent Harness

- Harness 是產品、模型、工具與 Runtime 之間的控制層。它規定 Agent 在什麼條件下理解任務、選擇能力並採取行動。
- 核心題目包括輸入正規化、Context 組裝、指令與能力配置、信任邊界、工具風險、Permission、Human Approval、執行中輸入、取消、追問與回饋。
- 閱讀可從 [Agent Harness 總覽](./agent-harness/README.md)開始。

### Agent Runtime

- Runtime 是 Harness 使用的運行時核心，負責把一次 Agent Run 推進到合法的終止狀態。
- 預計涵蓋 Execution Identity、模型串流、工具排程、重試、錯誤回灌、Checkpoint、Interrupt、Resume、對話修復、Idempotency、Side-effect Ledger、Reconciliation、Compensation、Runtime Event、Tracing、Metrics 與上線門檻。

## 完整的 Harness 與 Runtime 交接流程

~~~text
產品接收互動
  → Harness 正規化輸入
  → Harness 組裝 Context、規則與能力
  → Runtime 執行模型與工具迴圈
  → Runtime 向 Harness 請求政策或人工決策
  → Runtime 產生訊息、事件與終止結果
  → Harness 顯示結果並承接下一次互動
~~~

這是責任邊界，不是目錄邊界。真實程式可能把兩邊寫在同一個模組裡；判斷時應追問的是，每個決策與狀態轉移究竟由誰持有。

## Engineering 標記

區分四種狀態：

| 狀態 | 意義 |
|---|---|
| 已觀察 | 已在拆解的 Claude Code 程式路徑中確認 |
| 已接入 | 已連上 Chat Gun 的正式執行路徑 |
| 模組可用 | Primitive 或模組已存在，正式接線仍不完整 |
| 規劃中 | 仍需實作與驗證的設計方案 |

類別、Schema 或單元測試存在，不等於端到端 Runtime 能力已經成立。

## License

本專案採用 [MIT License](./LICENSE)。Copyright (c) 2026 Hsien Wei。
