# 幂等与重放识别

[English](./02-idempotency-and-replay-identity.md) | [繁體中文](./02-idempotency-and-replay-identity-zh-TW.md) | [简体中文](./02-idempotency-and-replay-identity-zh-CN.md)

> 以下内容所提到的
> CCB 为 [claude-code-best](https://github.com/claude-code-best/claude-code) 的简称。
> Chat Gun 为 [chat-gun](https://github.com/HsienW/chat-gun)。

[回到导览](./README-zh-CN.md) · 前一篇：[Checkpoint 与恢复数据](./01-checkpoints-and-recovery-state-zh-CN.md) · 下一篇：[效果账本与提交边界](./03-effect-ledger-and-commit-boundaries-zh-CN.md)

> 用户送出一次操作，连接中断后又重送。Runtime 建立新 Run，模型也产生新的 Tool Call ID；外部系统看见的业务要求却相同。若只比较 Run ID，系统可能再次执行；若只比较工具名称，又会把两次合法的不同操作合并。

**去重需要先定义「相同」的范围。** 请求、逻辑调用、实体 Attempt 与业务效果有不同身份，重用条件也随之改变。

## 五种身份各自回答什么

| 身份 | 回答的问题 | 不能取代的判断 |
|---|---|---|
| Request Dedup Key | 同一身份是否重送同一路由的请求？ | 工具是否产生相同业务效果？ |
| Replay Key | 是否重放同一 Run 中的指定逻辑调用？ | 新 Run 是否重做同一业务操作？ |
| Tool Execution ID | 哪份持久工具执行记录？ | 实际派送了几次？ |
| Attempt ID | 哪一次实体尝试？ | 多次尝试是否应合并成同一效果？ |
| Business Effect Key | 哪个可信 scope 中的业务效果？ | 外部服务是否已接受同一幂等键？ |

这些识别码需要可查回的关联。让所有字段都使用同一个随机值，会失去辨别逻辑调用与实体重试的能力；只增加更多 ID，也无法自动建立去重契约。

## Chat Gun 的键如何组成

本轮补查安装的 `@gun-ai/harness-kernel`，版本为 `0.1.0-alpha.1`。`identity.ts` 提供哈希与 UUID dependencies，kernel 保存组成规则；查读 `src/side-effect.ts` 与包 export 使用的 `dist/side-effect.js`，可确认以下字段。

Replay Key 的输入顺序为：

~~~text
runId
stepId
logicalToolCallId（缺少時使用 toolCallId）
callIndex
toolName
toolVersion
~~~

Kernel 验证必要字符串与非负安全整数 `callIndex`，再将这些值交给 wrapper 的 hash。Chat Gun wrapper 对数组的 JSON 表示计算 SHA256。

Request Dedup Key 使用 tenant、principal、route namespace 与 client key。Attempt identity 则保留 Tool Execution ID、正整数 `executionAttempt`，另产生 UUID。这些 helper 提供识别规则，入口是否采用及如何验证 client key，仍由实际 consumer 决定。

## 同一重放键与不同参数

Chat Gun runner 读到既有 Replay Key 后，先比较 `requestHash`。相同 key 配上不同 hash，返回 `SIDE_EFFECT_REPLAY_CONFLICT`；既有记录未 committed 时，返回需要回查，而非再次调用工具。

这个分支保护的是已记录的逻辑调用。`requestHash` 如何涵盖 canonical input、哪些字段应稳定化，仍需对回建立它的 Pipeline 与工具契约。若上游把不同语义的要求算成相同 hash，runner 无法从 hash 值还原遗失的差异。

同样地，模型产生新 ID 只表示另一份提案；系统仍应核对它是否指向同一业务效果。模型重新叙述要求，不能清除先前 unknown 的效果记录。

## 业务效果的去重范围

`deriveBusinessEffectKey(input, scope)` 由 descriptor 定义，再经 wrapper 哈希。数据库唯一约束使用：

~~~text
tenant_id + scope_id + business_effect_key
~~~

因此新 Run 虽然改变 Replay Key，仍可能由相同业务键关联到既有效果。业务键应依操作语义设计，例如可信订单与指定退款意图；这是教学例子，没有宣称来源库提供退款工具。

金额、资源版本、对象与重复操作是否允许，都会影响键的设计。键太粗会阻止合法的新操作，键太细则可能让重送变成新效果。Scope 也必须来自可信 Context，不能直接采用模型填入的 tenant 名称。

数据库另外约束外部 system namespace 与 operation ID 的配对及唯一性。这能维护本地已取得的外部关联；外部 API 是否接受并持久去重同一操作，仍需要服务端契约。

### Production MCP 的业务键范例

`production-runtime-tool-descriptors.ts` 对查读的非只读 MCP 工具，以 server name、tool name、tool version、tenant、scope 与 input 建立 canonical JSON，作为业务键的原始值。Canonicalizer 递归排序对象键，保留数组元素顺序。

这个规则可以让对象字段排列不同的相同输入取得稳定表示；它也保留了明确限制：工具版本或输入值改变，键就可能改变。两份业务上相同的要求若使用不同默认值、不同时间字符串或不同表示法，仍需工具自行定义语义规范化；反过来，业务允许再次执行完全相同参数时，也不能只靠参数相同判断为重送。

Pipeline 建立 runner identity 的查读区段则使用 canonical Tool Call ID，并将 `callIndex` 设为 `0`。Kernel 支持 call index，不代表这个 consumer 已为同名调用分配递增索引；此入口还依赖 Tool Call ID 的正确建立与配对。

## TTL 到期之后

`PgIdempotencyGuard` 保存 namespace、resource key、version 与 locked／completed／failed 状态。Kernel 将键序列化为 `namespace:resourceKey:vversion`，这套记录与 runner 的 Replay Key／效果 Ledger 分开使用。

Guard 可删除 failed 记录后重新取得，也可在过期条件成立时尝试回收。TTL 只限制本地记录有效期；远端操作可能仍在执行，或已提交而本地未补记。

在查读方法中，完成与失败更新依序列化 key 定位，没有在该更新条件中核对 owner token。若应用允许 TTL 回收后由另一执行者接手，就要检查旧持有者晚到的更新如何受控。这是需要消费端与并行情境验证的窗口，不能只因存在 Guard 便宣称具备跨 Worker fencing。

## 已提交结果何时可以重用

Ledger committed 让 Runtime 知道已有提交记录，模型需要的内容则可能放在 Result Reference。`PgResultReferenceStore.resolve()` 会检查 scope 兼容、tool version、cache state 与 `policy.isReusable()`，通过后才调用 `resolveResultRef()` 取得 payload。

| 情况 | 查读路径的处置方向 | 应保留的事实 |
|---|---|---|
| 相同重放键、不同 request hash | 回报 conflict。 | 原操作与新参数冲突。 |
| 既有执行尚未 committed | 回报需要回查。 | 尚无可直接重用的 committed 结果。 |
| Scope／版本不符或结果不可重用 | 拒绝结果重用。 | 既有效果仍可能已提交。 |
| Payload 缺失 | 无法交付既有结果。 | 内容遗失不等于效果不存在。 |
| 结果符合重用条件 | 返回既有结果。 | 结果的来源为 cache 或已回查数据。 |

结果重用是数据访问与兼容性判断，也仍受产品授权要求约束。不能把 cache 可读直接扩张成新操作的执行授权。

## CCB 的 ID 对照

CCB 的 tool-use／tool-result ID 支持消息配对，StreamingToolExecutor 也以工具关联收集结果。它们适合辨认结果属于哪份提案；本轮没有从这些 ID 找到与 Chat Gun 相同的跨 Run 业务效果唯一约束。

若要替 CCB 中某个工具建立外部幂等，仍需查该工具的 adapter、外部 API 与持久记录。Transcript 中出现一次成功消息，只是可用证据的一部分。

## 如何验证重放识别

- 同一 Replay Key 改变参数 hash，确认返回冲突且不派送。
- 新 Run 使用相同业务键，核对既有 committed 效果与结果重用条件。
- 让 payload 不可取得、scope 改变或版本不符，确认没有以内容缺失触发重做。
- 在 TTL 回收后交错新旧持有者的完成更新，核对 owner 与外部效果记录。
- 对外部幂等键测试保存期限及同键不同参数的服务端行为。

本轮读取识别、Guard、Ledger 与 Result Reference 实现，未执行外部 API 或多 Worker 重送测试。

## 本篇的工程判断

我会先写清楚业务上允许重复的操作，再设计键。重试应增加 Attempt 证据，重放应查回原逻辑调用，新业务操作则需要自己的语义身份；这些关联能让故障后的处理保有依据。

## 实现参照与适用范围

- Chat Gun：`runtime/side-effect/identity.ts`、`tool-execution-runner.ts`、`business-effect-ledger.ts`、`result-reference-store.ts` 与 `runtime/idempotency/idempotency-guard.ts`，路径相对 `backend/src/`。
- Chat Gun production MCP：`backend/src/tools/production-runtime-tool-descriptors.ts` 的 canonical input 与业务键；`backend/src/runtime/tool-dispatch/pipeline.ts` 的 identity 建立。
- Kernel：`@gun-ai/harness-kernel` 的 `src/side-effect.ts`、`dist/side-effect.js` 与 `src/idempotency.ts`。
- [来源与文件指纹](./README-zh-CN.md#来源与查证状态)记录包及本地文件；本文没有建立对所有外部服务通用的 exactly-once 保证。

## 接续阅读

下一篇[效果账本与提交边界](./03-effect-ledger-and-commit-boundaries-zh-CN.md)沿实际派送顺序，追查外部效果与本地提交之间的故障窗口。
