# Multi-Agent Harness 基础篇

[English](./README.md) | [繁體中文](./README-zh-TW.md) | [简体中文](./README-zh-CN.md)

一封任务消息要让 Worker 知道在哪个工作区操作、向谁回复，也要让负责验收的人能识别交付物。以下以 Munder Difflin 的 Hive 协作协议为例。

1. [如何把工作交给另一个 Agent？](./01-delegation-contract-zh-CN.md)　从一次 README 检查入手，确定收件人、工作范围和报告要求。
2. [收到回信后，谁能判定交付完成？](./02-delivery-and-acceptance-zh-CN.md)　核对实现和验证结果是否指向同一版本，判断还需要补充哪些证据。
3. [一项工作该拆给哪些 Worker？](./03-work-decomposition-zh-CN.md)　回到分派前，判断哪些工作可以并行、谁需要等待上游交付，以及什么时候一个 Worker 就够了。
4. [要求变了，哪些委派仍然有效？](./04-revising-delegated-work-zh-CN.md)　跟踪一次范围变更，核对受影响的委派、可继续使用的产物及需要补测的版本。

消息与回复的关联方式参考 Munder Difflin 的固定版本。各篇中的任务分工、版本 B 与 C、改派和验收判断均为教学场景。[版本 96ae43e](https://github.com/HarnessMD/munder-difflin/commit/96ae43e3292447f68af6904afb353f6f5bdaa412) · [Hive 协作协议](https://github.com/HarnessMD/munder-difflin/blob/96ae43e3292447f68af6904afb353f6f5bdaa412/HIVE.md)

[返回 Multi-Agent Harness（繁体中文）](../README-zh-TW.md)
