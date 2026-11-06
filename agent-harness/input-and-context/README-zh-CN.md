# Input and Context

[English](./README.md) | [繁體中文](./README-zh-TW.md) | [简体中文](./README-zh-CN.md)

本区接在[基础篇](../foundations/README-zh-CN.md)之后，追踪 Harness 如何接收任务、决定模型能看见的材料，并保留指令与数据的界线。请依序阅读；各篇区分已读到的代码路径、已接入正式入口的能力与作者的设计判断，尚需实际执行证据的部分，也在文中标明。

## 阅读顺序

| 文章 | 要回答的问题与篇章边界 | 主要参照 |
|---|---|---|
| [01. 输入正规化](./01-input-normalization-zh-CN.md) | 模型执行前，哪些内容会成为任务输入？范围限於正规化结果与来源。 | claude-code-best 的 `processUserInput`；Chat Gun 的 `normalized-agent-input.ts`、`read-normalized-agent-input.ts`、`execution-composition-root.ts`。 |
| [02. Context 组装](./02-context-assembly-zh-CN.md) | 哪些区块进入模型调用，顺序与容量如何决定？ | claude-code-best 的 query 准备流程；Chat Gun 的 `context-integration.ts`、`context-source.ts`、`agent-context-assembler.ts`。 |
| [03. Context 信任边界](./03-context-trust-boundary-zh-CN.md) | 哪些来源可下指令，哪些只能当任务数据？ | Chat Gun 的来源分类与当前任务优先序；基础篇整理的指令与工具输出边界。 |
| [04. Memory 与压缩](./04-memory-and-compaction-zh-CN.md) | 记忆提取与压缩如何延续任务，又不盖过本次要求？ | Chat Gun 的 Memory recall、容量、压缩与 manifest；claude-code-best 各自独立的 compacting 路径。 |
| [05. 从输入到 Context](./05-case-input-to-context-zh-CN.md) | 能否沿着一次请求，核对模型实际看见的内容？ | Chat Gun chatbot 入口，逐项标出纳入、排除与失败位置。 |

部署的端到端 Memory 行为取决於 provider 配置与相关 Agent 入口；实际执行则需要可重现的请求及其 Context manifest。容量优先序只能说明取舍，不能单凭它宣称已挡下 Prompt Injection。

## 证据与适用范围

「已观察」指可从源代码定位的行为；「已接入」还需要正式入口的调用证据；设计建议则需要实际产品接线与验证。代码库与版本共同限定每项判断的适用范围。
