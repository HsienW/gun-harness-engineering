# Checkpoint 与恢复数据

[English](./01-checkpoints-and-recovery-state.md) | [繁體中文](./01-checkpoints-and-recovery-state-zh-TW.md) | [简体中文](./01-checkpoints-and-recovery-state-zh-CN.md)

> 以下内容所提到的
> CCB 为 [claude-code-best](https://github.com/claude-code-best/claude-code) 的简称。
> Chat Gun 为 [chat-gun](https://github.com/HsienW/chat-gun)。

[回到导览](./README-zh-CN.md) · 下一篇：[幂等与重放识别](./02-idempotency-and-replay-identity-zh-CN.md)

> 程序停止前，工具已经送出写入；重启后读回的 Checkpoint 却仍显示这个节点尚待执行。若直接从节点位置重跑，外部系统可能收到第二次操作。若只恢复画面上的对话，Runtime 又可能遗失尚待确认的操作与版本条件。

**恢复需要交叉核对执行位置、等待契约与外部效果证据。** Checkpoint 提供其中一份状态，是否可以续行，还要沿着 Manifest、Task／Step 与 Ledger 查明。

## 需要读回哪些数据

Graph 可以记得下一个节点，Task repository 可以记得工作仍在 running，外部系统则可能早已提交。这些数据记录不同事实，程序停止的位置会让它们暂时不一致。

| 数据 | 主要用途 | 无法单独回答的问题 |
|---|---|---|
| Checkpoint | 保存 Graph values、消息与待处理节点。 | 外部写入是否已完成？ |
| Interrupt Manifest | 指定等待、回复 Schema、期限与执行关联。 | 回复消费后是否成功进入 Graph？ |
| Execution Manifest | 保存 Graph 设置与相关 Schema 版本。 | 新部署是否已具备数据迁移器？ |
| Task／Step | 记录工作与步骤的持久状态。 | 外部成功但本地补记失败时，真实效果为何？ |
| Ledger／Result Reference | 查找执行尝试、效果状态与既有结果。 | 遗失的外部回应是否可由其他来源补回？ |

这份表格整理责任，没有要求所有 Runtime 使用相同存储结构。恢复时应先辨认哪份数据对哪个事实有权威，再处理矛盾，不能以最后写入的时间取代语义判断。

## Chat Gun 的恢复入口

`createConversationRecovery().recover()` 从 `interruptId` 寻找 Manifest，再核对 thread、Run、Task 与 scope。回复已提供时，还会检查 Manifest 是否 waiting，以及期限是否有效。

接着读取 Checkpoint。找不到时返回 `CHECKPOINT_NOT_FOUND` 的人工介入结果；读到数据后才进入历史整理、持久证据加载与执行位置分类。这条入口要求一份可找到的 Interrupt Manifest，不能由函数名称推论它能接手所有没有等待记录的程序故障。

概念顺序如下，省略个别拒绝出口：

~~~text
寻找 Manifest 并核对关联
  → 读取 Checkpoint
  → 核对版本并整理历史
  → 载入 Task／Step／Ledger 证据
  → 分类最後执行位置
  → 回传续行、等待、回查或人工处理结果
~~~

源代码可确认这些调用顺序；`loadDurableEvidence` 与 Reconciler 的实际接线由 dependencies 提供，必须另外查证。

## 版本兼容性先于历史重用

Execution Manifest 保存 Graph ID、Graph 设置哈希与 Schema 版本。`classifyExecutionManifestCompatibility()` 会比较 Manifest major version、Graph ID、设置哈希及各 Schema 版本。

| 分类 | 查读实现的判断方向 | 后续责任 |
|---|---|---|
| compatible | 关键设置与 Schema 比较通过。 | 仍须检查历史及执行证据。 |
| migratable | Schema 有差异，但未触及比较中的 major 差异。 | 需要具体迁移与验证。 |
| incompatible | Graph、设置或受检查的 major version 不符。 | 停止直接重用。 |

这个分类器没有执行 migration。Sanitizer 对需要迁移的数据可返回停放原因，不能把 `migratable` 显示成已完成自动升级。部署版本也应依实际比较字段理解，不能假设 Manifest 中每个字段都已成为拒绝条件。

## 历史整理保留什么

恢复的消息可能包含只有开头的 Assistant Message、没有对应结果的 Tool Call，或旧 executor 留下的重复结果。这些片段进入下一轮模型之前，需要先查明是否仍可配对。

Chat Gun `sanitizeRecoveryHistory()` 具有 valid、sanitized 与 parked 三种结果。它也保留未配对调用、未配对结果、重复结果、不完整参数、版本不兼容与无法证明历史等 reason code。

这些 code 代表不同处理分支，不能理解成所有异常都直接删除后续行。Sanitized 结果保留整理后的 history 与原因；parked 结果交由人工处理，`recover()` 不继续消费回复。

历史整理的目的在于产生可使用的对话与配对。删除一份旧 tool result，不会撤销它曾触发的外部写入，也不能把 Ledger 的 unknown 改成未提交。

## 最后执行位置的判断顺序

`classifyLastExecutionPoint()` 使用 Task、可选 Step、Ledger、Checkpoint 与 Manifest 证据。其判断有顺序：

1. Checkpoint 或指定 Task／Step 状态符合终止条件，先分类为 terminal。
2. Ledger 为 committed 或 unknown，采用对应分类。
3. Manifest waiting 时分类为 waiting_user。
4. 符合尚未开始的组合时分类为 not_started；其他执行中证据可分类为 executing。
5. 仍无法归类时保留 unknown。

Terminal 优先会阻止一般续行，后续效果查证仍可作为独立工作保留。分类器能接受「Checkpoint 不存在」的组合；但目前 `recover()` 在调用它之前就会拒绝缺失的 Checkpoint，两个层次的可达路径应分开阅读。

`executing` 只表示读到执行中证据。实际接手者仍需核对 ownership、原 Worker 是否存活与工具的 replay 路径，才能决定如何继续。

## Resume 消费之后的窗口

在 waiting_user 分支，Recovery 验证 response Schema、以已保存事件的最大序号初始化 allocator，再尝试消费指定等待。Confirmation 另透过提供的 authorizer 消费；缺少它时不直接批准。

消费成功后，Checkpoint Adapter 依序执行：

~~~ts
await graph.updateState(config, { messages: sanitizedHistory });
return graph.invoke(new Command({ resume: response }), config);
~~~

这是查读 Adapter 的核心顺序。Manifest 消费、历史更新与 Graph invoke 分属不同调用，程序可在任两步之间停止。回复只能消费一次，也仍可能遇到「已消费但尚未接手」的空窗，需要恢复接手记录或其他可证明的补接机制。

Adapter 没有决定底层 Checkpointer 的持久性。Deep Researcher 查读的位置使用 `MemorySaver`，单凭它无法保证程序重启后仍可找回状态；部署采用哪种 Checkpointer，须由实际 Graph 与环境设置确认。

## CCB 的 Session 恢复对照

CCB `QueryEngine.ts` 依 `isSessionPersistenceDisabled()` 决定是否保存 transcript。查读区段有等待 `recordTranscript()` 完成的路径，也有 `void recordTranscript()` 的异步写入；consumer 是否等到写入完成会影响故障时可读回的范围。

`sessionStorage.ts` 提供 transcript 记录与加载 helper；`sessionRestore.ts` 则整理恢复后的 messages、file history、agent 设置与其他 session 状态。这些是 Session 恢复的具体接点，无法单独证明每份工具外部效果都有 durable Ledger 或恢复交易。

对照两个来源时，可以问它们如何恢复可供推理的历史，再分别追踪工具效果。Session messages、Graph Checkpoint 与外部业务记录各有保存契约。

### Transcript 如何找回一条对话

CCB `loadTranscriptFromFile()` 在 JSONL 路径先调用 `loadTranscriptFile()`，找出 leaf message，再由 leaf 向 root 建立 conversation chain。没有消息或没有有效链时会抛错，并非读入文件中的所有列就直接接续。

这说明恢复还有结构条件：消息存在、父子关联可追溯，以及选定哪个分支是三件事。`recordTranscript()` 也会先辨认已记录消息，保留前缀中的 chain participant，再插入新消息链。部分数据仍在文件里，不代表它一定属于目前恢复的可达分支。

因此验证 Session 恢复时，除了查看消息数量，也要核对 leaf、父子链、工具配对及 compact 后的有效历史。这些结构恢复仍没有替外部操作建立未提交证据。

## 如何验证恢复数据

- 分别让 Manifest、Checkpoint 与 Ledger 缺失，核对每种组合的出口及原因。
- 注入部分消息、重复结果与不兼容版本，确认 parked 时没有消费回复。
- 在 Resume 消费后、历史更新后与 Graph invoke 前停止程序，核对重新接手方式。
- 对同一部署重启程序，确认 Checkpointer 实际可读回；再测不同 Worker 是否读到同一份状态。

上述是验证设计。本轮读取 Recovery 实现及既有测试安排，未执行程序重启、数据库或跨 Worker 演练。

## 本篇的工程判断

我会先列出恢复需要的事实及其权威来源，再决定读取顺序。数据彼此矛盾时，保留原因与未解效果，能让后续回查知道缺少什么；直接重跑节点会让这些矛盾变成第二次操作。

## 实现参照与适用范围

- Chat Gun：`runtime/recovery/conversation-recovery.ts`、`langgraph-checkpoint-adapter.ts`、`last-execution-point.ts`、`execution-manifest.ts` 与 `recovery-sanitizer.ts`，路径相对 `backend/src/`。
- CCB：`src/QueryEngine.ts`、`src/utils/sessionStorage.ts` 与 `src/utils/sessionRestore.ts`。
- [来源与文件指纹](./README-zh-CN.md#来源与查证状态)记录本地查读范围；本文没有宣称通用 Recovery 入口已在所有正式 Graph 启用。

## 接续阅读

下一篇[幂等与重放识别](./02-idempotency-and-replay-identity-zh-CN.md)追查恢复之后如何辨认同一次调用、下一个 Attempt 与相同业务效果。
