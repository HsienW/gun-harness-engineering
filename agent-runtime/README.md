# Agent Runtime

[English](./README.md) | [繁體中文](./README-zh-TW.md) | [简体中文](./README-zh-CN.md)

This section covers the Agent's execution core, from establishing execution identity and advancing Run state through model and tool calls, waits for external decisions, side effects, and failures, to a verifiable terminal result.

The Runtime receives input, Context, capabilities, and policy conditions prepared by the [Agent Harness](../agent-harness/README.md) and follows those product rules. When execution needs authorization, human confirmation, or additional user input, the Runtime returns the question to the Harness through an explicit Decision Port, then continues or stops according to the structured decision.

## Reading Structure

| Category | Scope |
|---|---|
| [Foundations](./foundations/README-zh-TW.md) | Runtime definition, Execution Identity and state authority, Run Loop, persistence and recovery, design principles, and a complete Run. |
| [Execution Lifecycle](./execution-lifecycle/README-zh-TW.md) | Run admission and creation, execution ownership and replacement, deadlines and cancellation, waiting and resumption, terminal cleanup, and lifecycle races. |
| [Model and Tool Execution](./model-and-tool-execution/README-zh-TW.md) | Model calls and Provider fallback, stream assembly, tool dispatch and concurrent scheduling, backpressure and capacity, result feedback, and a model and tool round trip. |
| [Effects and Recovery](./effects-and-recovery/README-zh-TW.md) | Checkpoints and recovery data, idempotency and replay identity, effect ledgers and commit boundaries, reconciliation and retry, compensation, and an ambiguous commit recovery case. |
| [Events and Operations](./events-and-operations/README-zh-TW.md) | Event identity and ordering, persistence and delivery, tracing, audit and data boundaries, metrics and health, fault validation and release gates, and investigating a missed terminal event. |

## The Runtime's Responsibility

~~~text
Harness 交付 Execution Request
  → Runtime 驗證執行契約並建立 Run Identity
  → Runtime 推進模型、工具、等待與恢復狀態
  → Runtime 回報事件、證據與 Terminal Result
  → Harness 將結果轉成產品互動
~~~

Implementations can use different classes and state enumerations for this flow. An imperative query loop, StateGraph, or remote Agent Server can serve as the Runtime. To identify its responsibilities, trace who owns execution state, valid transitions, stopping conditions, and recovery evidence.

## Division of Responsibilities with the Harness

| Question | Primary authority |
|---|---|
| How user input is normalized and which Context is visible | Harness |
| Capability sets, authorization policies, and human confirmation rules | Harness |
| How a Run starts, waits, is canceled, and reaches a terminal state | Runtime |
| How model and tool execution advances within budget | Runtime |
| Whether an external operation actually committed | External systems and Side-effect Evidence |
| How execution results are presented to the user | Harness |

The Runtime enforces rules supplied by the Harness and can stop to request an external decision when one is missing. Trusted identity, authorization, and evidence of external commits must come from authoritative sources. Prompt text, tool parameters, and UI wording cannot establish those facts on their own.

## Writing about Evidence

This section follows the knowledge base's Engineering markers to distinguish observed source code, capabilities integrated into official entry points, Primitives available only as modules or tests, and designs awaiting implementation or deployment verification. Claims about cross-process recovery, idempotent external side effects, or production availability require evidence beyond the existence of a class, test case, in-memory Checkpointer, or successful response.
