# Multi-Agent Runtime 基础篇

[English](./README.md) | [繁體中文](./README-zh-TW.md) | [简体中文](./README-zh-CN.md)

Munder Difflin 的主进程通过伪终端（PTY）启动 Worker CLI。Hive 管理任务分派和消息，CLI 则在自己的回合中调用模型采样，并把工具结果传回模型。要追踪一次执行，可以先根据主进程记录的身份找到 Worker，再沿着消息找到待处理的工作。[平台架构](https://github.com/HarnessMD/munder-difflin/blob/96ae43e3292447f68af6904afb353f6f5bdaa412/docs/ARCHITECTURE.md)

1. [Worker 重启后，平台还在和谁通信？](./01-worker-identity-zh-CN.md)　辨认五种 ID，追踪复用同一 PTY ID 的新进程，以及启动参数被截断的故障。
2. [信已送达，Worker 何时会处理？](./02-mailbox-and-cli-progress-zh-CN.md)　分别观察 Router 投递、CLI 收件、回合结束和 Worker 回复。
3. [平台重启后，凭什么接着执行原任务？](./03-recovery-and-state-evidence-zh-CN.md)　查看角色记忆、Transcript、CLI Session 和 Task，判断还缺哪些证据。

源代码、设计文档和修复记录都取自[同一版本 96ae43e](https://github.com/HarnessMD/munder-difflin/commit/96ae43e3292447f68af6904afb353f6f5bdaa412)。PTY 回调和 Session 登记可以查源代码，Stop hook 的预期行为来自 Hive 设计文档。各篇列出的场景仍待实测。

[返回 Multi-Agent Runtime（繁体中文）](../README-zh-TW.md)
