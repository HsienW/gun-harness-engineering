<p align="center">
  <img src="./assets/00-gun-harness-engineering-logo.png" alt="chat-gun" width="1024"/>
</p>

<p align="center">
  <a href="./README.md">English</a> |
  <a href="./README-zh-TW.md">繁體中文</a> |
  <a href="./README-zh-CN.md">简体中文</a>
</p>

---

## Gun Harness 是什么?

这是一套讨论 AI Agent 在真实产品中受到约束、完成执行、故障恢复的知识库

- 这些内容起源自个人对于 [chat-gun](https://github.com/HsienW/chat-gun) 的实战加上对 [claude-code-best/claude-code](https://github.com/claude-code-best/claude-code) 的研究与思考。
- 本库不讨论各种以 Agent Infra 为出发点的 Harness 论文，而是聚焦 Harness 在业务落地中的两个核心问题
  - 从产品实践视角如何定位 Harness 的责任与能力
  - 业务落地时 Harness 跟 Runtime 的边界，以及各自负责什么任务

### 首先！不用技术名词，你能说清楚 Harness 跟 Runtime 是什么吗?

- 如果你回答不出来或答案有模糊点，先看[这篇](https://github.com/HsienW/ai-agent-coding-solution-kit/blob/master/agent-thoughts/ai-agent-basics/01-AI-Harness-Runtime-Jargon-Free.zh-CN.md) 能帮助你更清晰地区分两者！

## 业务视角看 Harness 与 Runtime

- **Agent** = Harness + LLM 这是目前业界公认说法，在这基础上往下拆分，如下图:

<p>
  <img src="./assets/01-gun-harness-pyramid.webp" alt="gun-harness-pyramid" width="768" />
</p>

图中由下而上可以看到四个部分 (最上层绑定的领域是跟随业务来替换的，越往下越是 Agent 的通用能力)

1. Infra: 这是属于模型范围，例如: GPT-5 / GPT-6 和 Claude Opus / Sonnet 等系列都是大众熟悉的模型。
2. Harness: 是模型的控制层，决定能力跟大方向的控制器，例如: 构建与编排层（Prompt / Workflow）、连接层（API / MCP 等协议、能力层（Skills / Tools） 都在这层。
3. Runtime: 是大方向已经被控制器定好之后，真正运行时的管控，例如: 沙箱环境、状态与记忆、重试机制、节流、权限治理、Trace 观测都在这。
4. Business: 是具体业务在前三层之上叠加不同 Domain 落地的范围，其实也是现在 FDE 在做的事，例如: 教育、金融、航空等等不同业务领域的 Agent 落地。

**因为 Business 强绑定业务、而 Infra 归属模型能力跟模型训练范畴，所以本库只会聚焦在以下两个的主线:**

- **Harness**： 产品如何决定 Agent 看见什么、可以使用哪些能力、行动如何取得授权，以及人如何介入、跟追踪路径。
- **Runtime**： 一次 Agent Run 如何推进状态、执行模型与工具、处理外部副作用、从失败恢复，并留下可供运营者检查的证据。

## 为什么建立这个知识库

Agent 架构常被写成一张组件清单：Prompt、Memory、Tool、Workflow、Tracing、Evaluation。这类清单没有回答谁拥有决策权、执行在哪里跨越信任边界，也没有说明工具其实已成功、回应却遗失时该怎么办。

本库从几个更接近实际运行的问题出发：

- 哪些责任属于 Harness，哪些属于 Runtime？
- 用户输入、历史记忆与外部数据冲突时，谁有较高权威？
- 谁可以批准一次会改变外部世界的行动？
- 工具失败如何回灌模型，又不破坏对话结构？
- 哪组识别码可以串起用户请求、Run、模型请求、工具调用与副作用？
- 程序崩溃后，哪些状态可以安全恢复？
- 哪些能力已经走过正式执行路径，哪些仍停在模块或规划？

## 知识库地图

~~~text
gun-harness-engineering/
├─ README.md
├─ README-zh-TW.md
├─ README-zh-CN.md
├─ agent-harness/
│  ├─ foundations/
│  ├─ input-and-context/
│  ├─ behavior-and-capabilities/
│  ├─ action-governance/
│  └─ interaction-and-feedback/
└─ agent-runtime/
   ├─ foundations/
   ├─ execution-lifecycle/
   ├─ model-and-tool-execution/
   ├─ effects-and-recovery/
   └─ events-and-operations/
~~~

### Agent Harness

- Harness 是产品、模型、工具与 Runtime 之间的控制层。它规定 Agent 在什么条件下理解任务、选择能力并采取行动。
- 核心题目包括输入规范化、Context 组装、指令与能力配置、信任边界、工具风险、Permission、Human Approval、执行中输入、取消、追问与反馈。
- 阅读可从 [Agent Harness 总览](./agent-harness/README-zh-CN.md)开始。

| 分类 | 去阅读 |
|---|---|
| Foundations | [传送门](./agent-harness/foundations/README-zh-CN.md) |
| Input and Context | [传送门](./agent-harness/input-and-context/README-zh-CN.md) |
| Behavior and Capabilities | [传送门](./agent-harness/behavior-and-capabilities/README-zh-CN.md) |
| Action Governance | [传送门](./agent-harness/action-governance/README-zh-CN.md) |
| Interaction and Feedback | [传送门](./agent-harness/interaction-and-feedback/README-zh-CN.md) |

### Agent Runtime

- Runtime 是 Harness 使用的运行时核心，负责把一次 Agent Run 推进到合法的终止状态。
- 目前涵盖 Execution Identity、模型流式传输、工具调度、重试、错误回灌、Checkpoint、Interrupt、Resume、对话修复、Idempotency、Side-effect Ledger、Reconciliation、Compensation、Runtime Event、Tracing、Metrics 与上线门槛。
- 阅读可从 [Agent Runtime 总览](./agent-runtime/README-zh-CN.md)开始。

| 分类 | 去阅读 |
|---|---|
| Foundations | [传送门](./agent-runtime/foundations/README-zh-CN.md) |
| Execution Lifecycle | [传送门](./agent-runtime/execution-lifecycle/README-zh-CN.md) |
| Model and Tool Execution | [传送门](./agent-runtime/model-and-tool-execution/README-zh-CN.md) |
| Effects and Recovery | [传送门](./agent-runtime/effects-and-recovery/README-zh-CN.md) |
| Events and Operations | [传送门](./agent-runtime/events-and-operations/README-zh-CN.md) |

## 完整的 Harness 与 Runtime 交接流程

~~~text
產品接收互動
  → Harness 正規化輸入
  → Harness 組裝 Context、規則與能力
  → Runtime 執行模型與工具迴圈
  → Runtime 向 Harness 請求政策或人工決策
  → Runtime 產生訊息、事件與終止結果
  → Harness 顯示結果並承接下一次互動
~~~

这是责任边界，不是目录边界。真实程序可能把两边写在同一个模块里；判断时应追问的是，每个决策与状态转移究竟由谁持有。

## Engineering 标记

区分四种状态：

| 状态 | 意义 |
|---|---|
| 已观察 | 已在拆解的 Claude Code 程序路径中确认 |
| 已接入 | 已连上 Chat Gun 的正式执行路径 |
| 模块可用 | Primitive 或模块已存在，正式接线仍不完整 |
| 规划中 | 仍需实现与验证的设计方案 |

类别、Schema 或单元测试存在，不等于端到端 Runtime 能力已经成立。

## License

本项目采用 [MIT License](./LICENSE)。Copyright (c) 2026 Hsien Wei。
