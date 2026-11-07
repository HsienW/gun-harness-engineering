# Agent Harness 基础篇

[English](./README.md) | [繁體中文](./README-zh-TW.md) | [简体中文](./README-zh-CN.md)



这五篇先定义 Harness 负责什麽，再追问决策如何进入执行，最后用一次文件修改任务核对整条路径。依序阅读，后一篇会把前一篇留下的问题推进到具体流程。



## 阅读顺序

1. [实际落地的 Agent Harness 是什麽](./01-what-is-agent-harness-zh-CN.md)：界定 Harness 在 Agent Product 与 Runtime 之间的责任；后续篇章沿用这里的名词。
2. [Harness 的控制层](./02-control-surfaces-zh-CN.md)：把定义拆成输入、Context、行为与能力、行动治理、互动与反馈五个可检查的决策面。
3. [Harness 与 Runtime 的边界](./03-harness-runtime-boundary-zh-CN.md)：追踪 Harness 交付哪些执行条件、Runtime 何时回来请求决策，以及状态与失败由谁负责。
4. [Agent Harness 设计原则](./04-design-principles-zh-CN.md)：从前述边界整理工程取舍，包括依任务选取 Context、区分指令与数据权威、检查行动，以及从正式入口验证规则。
5. [一次任务的完整走法](./05-one-turn-walkthrough-zh-CN.md)：用文件修改情境串起输入、Context、工具结果、授权、失败与完成证据，检查前四篇的责任分工如何运作。
6. 接着可读 [Input and Context](../input-and-context/README-zh-CN.md)，进一步展开输入规范化、模型可见材料、来源权威与 Memory。



* 文中的实现参照库，应分清已观察的程序路径、已接入入口的能力、仅有模组与设计提案；教学案例也不能当成正式环境的执行纪录。

  * [claude-code-best](https://github.com/claude-code-best/claude-code)
  * [chat-gun](https://github.com/HsienW/chat-gun)
