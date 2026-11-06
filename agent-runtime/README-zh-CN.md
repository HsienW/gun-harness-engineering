# Agent Runtime

[English](./README.md) | [繁體中文](./README-zh-TW.md) | [简体中文](./README-zh-CN.md)

本区整理 Agent 的执行核心：一次 Run 如何取得执行身份、推进状态、调用模型与工具、等待外部决策、处理副作用与故障，最后产生可验证的终止结果。

Runtime 接收 [Agent Harness](../agent-harness/README-zh-CN.md) 准备好的输入、Context、能力与策略条件，但不重新决定这些产品规则。执行途中若需要授权、人工确认或用户补充，Runtime 通过明确的 Decision Port 把问题交还 Harness，再依结构化决策继续或停止。

## 阅读架构

| 分类 | 范围 |
|---|---|
| [Foundations](./foundations/README-zh-CN.md) | Runtime 定义、Execution Identity 与状态权威、Run Loop、持久化与恢复、设计原则及一次完整 Run。 |
| [Execution Lifecycle](./execution-lifecycle/README-zh-CN.md) | Run 接纳与建立、执行归属与替换、期限与取消、等待恢复、终止清理及生命周期竞态。 |
| [Model and Tool Execution](./model-and-tool-execution/README-zh-CN.md) | 模型调用与 Provider 切换、流式传输组装、工具派送与并行调度、背压容量、结果回灌及一次模型与工具往返。 |
| [Effects and Recovery](./effects-and-recovery/README-zh-CN.md) | Checkpoint 与恢复数据、幂等与重放识别、效果账本与提交边界、回查重试、补偿及提交不明的恢复案例。 |
| [Events and Operations](./events-and-operations/README-zh-CN.md) | 事件身份与顺序、保存交付、追踪审计与数据边界、指标健康、故障验证与上线门槛及漏接终止事件的查证。 |

## Runtime 的责任位置

~~~text
Harness 交付 Execution Request
  → Runtime 验证执行契约并建立 Run Identity
  → Runtime 推进模型、工具、等待与恢复状态
  → Runtime 回报事件、证据与 Terminal Result
  → Harness 将结果转成产品互动
~~~

这条流程描述责任，不代表所有实现都使用相同类别或同一套状态枚举。Imperative query loop、StateGraph 与远端 Agent Server 都可以承担 Runtime，判断依据是谁持有执行状态、合法转移、停止条件与恢复证据。

## 与 Harness 的分工

| 问题 | 主要权威 |
|---|---|
| 用户输入如何规范化、哪些 Context 可见 | Harness |
| 能力集合、授权策略与人工确认规则 | Harness |
| Run 如何开始、等待、取消及收敛 | Runtime |
| 模型与工具如何依预算推进 | Runtime |
| 外部操作是否真正提交 | 外部系统与 Side-effect Evidence |
| 执行结果如何呈现给用户 | Harness |

Runtime 可以执行 Harness 已交付的规则，也可以在缺少决策时停止并请求外部判断。它不能从 Prompt、工具参数或 UI 文案自行创造可信身份、授权或外部提交事实。

## 证据与适用范围

本区沿用全库的 Engineering 标记，分开描述已观察的源代码、已接入正式入口的能力、只存在于模块或测试中的 Primitive，以及仍待实现或部署验证的设计。类别、测试案例、内存 Checkpointer 或成功回复都不能单独证明跨进程恢复、外部副作用幂等或生产环境可用。
