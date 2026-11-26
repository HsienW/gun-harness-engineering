# 如何把工作交给另一个 Agent？

[English](./01-delegation-contract.md) | [繁體中文](./01-delegation-contract-zh-TW.md) | [简体中文](./01-delegation-contract-zh-CN.md)

假设协调者要找出 README 中的一个失效链接。如果只交代“检查文档”，即使 Worker 回复“已检查”，接手的人也得猜它检查的是哪个版本。指定工作区、版本和文件，再要求回信附上行号和目标路径，另一个人就能重复这次检查。

| 分派时要说清什么 | 这次检查的约定 |
|---|---|
| 谁把工作交给谁 | 协调者创建 Task，指定负责的 Worker，并保留原始任务消息的 Message ID。 |
| 可以做什么 | Worker 读取指定版本的 README，按文件所在目录解析相对路径。如果需要修改文件，先报告，由协调者确认是否扩大范围。 |
| 如何报告结果 | Worker 通过 outbox 回信，用 `in_reply_to` 指向原消息，并附上版本、行号和失效路径。 |
| 谁来结案 | 协调者打开指定版本的文件，核对链接目标，再把验收结果记入 Task。 |

## Worker 收到协作规则了吗？

Worker 需要知道如何使用 inbox 和 outbox，才能读取任务并回复原消息。Munder Difflin 曾在 Windows 上出现过这种故障：Worker 启动成功，却缺少这段指令。协调者可以发送一条带标识的请求，检查 Worker 的回复是否指向原请求。启动参数在何处被截断，以及进程端能取得哪些证据，见 [Worker 启动与身份（繁体中文）](../../multi-agent-runtime/foundations/01-worker-identity-zh-TW.md)。

Hive 的设计区分了需要回答的请求和可以结束往返的通知。如果协调者更换收件人或要求检查另一个版本，分派任务时也应写明新的负责人和目标，让验收者知道哪些报告仍然适用。上表列出的是教学任务的约定，实际消息字段见 [Hive 协作协议](https://github.com/HarnessMD/munder-difflin/blob/96ae43e3292447f68af6904afb353f6f5bdaa412/HIVE.md)。

[返回本区基础篇](./README-zh-CN.md)
