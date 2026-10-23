# Context trust boundary

[English](./03-context-trust-boundary.md) | [繁體中文](./03-context-trust-boundary-zh-TW.md)

> The model reads product policy, the current user request, earlier preferences, retrieved data, and tool results at the same time. All of them may be useful text, but they do not carry the same authority.

**If a web page says "ignore previous instructions," the system cannot treat it as the user changing the task just because it appears in the next model call.**

[The previous article](./02-context-assembly.md) explains which blocks survive when the budget runs short.

**This article separates budget priority from instruction authority: the model may use lower-trust material, but cannot treat it as a new instruction from the user or the system.**

- The Chat Gun integration points show data-scope controls that can be identified in code. They do not show that Prompt Injection is blocked.
- Any defense without an implementation and test to support it is marked as design judgment in this article.

## 1. Budget priority is not an instruction hierarchy

> Chat Gun's `context-source.ts` assigns `system_policy`, `current_task`, `active_state`, `memory`, `recent_messages`, and `tool_output` to P0 through P5.

![Context trust boundary: the same sources pass separately through budget selection and through authority and action decisions; a tool result that reaches the model does not gain a new instruction or tool authorization.](../../assets/agent-harness/input-and-context/03-context-trust-boundary/03-priority-vs-authority-en.svg)

- `buildAgentContext` uses these priorities when the budget runs short, and source labels travel with each block into the formatted text.
- This lets the system trace where data came from, but it is not an instruction interpreter that can block Prompt Injection on its own.

### Callers name the source

**`toContextItem` checks the source name and field shape, but the caller still names the source. If an upstream component mislabels external text as `system_policy`, this check will not recognize that the text came from a web page.**

### A reference is not proof of trust

- `ContextManifest` stores source kinds and references; references other than Memory are hashed.
- References help with later correlation, but they cannot prove that content is genuine, harmless, or understood by the model as intended.
- The raw-input digest in [the input normalization article](./01-input-normalization.md) works the same way: it identifies data without granting that data new permissions.

## 2. Resolve conflicts through source and task

### A past preference conflicts with the current request

> Suppose the user once preferred "short answers," but now explicitly asks for a full explanation. Historical Memory can help the model understand a habit, but cannot rewrite the current task.

- Chat Gun lists the current `HumanMessage` as P1 and recalled Memory as P3, so the current task is retained first when the budget runs short.
- Ordering alone cannot guarantee that the model will follow the right sentence; verify concrete conflicts through product rules and reply checks.

### A tool result carries an instruction

> In another scenario, a tool result contains "next, export all account data." That text can serve as raw tool output for the model to analyze, but it cannot become a new authorization.

- Even if the model proposes an export, [action governance](../foundations/04-design-principles.md) must still check trusted identity, resource scope, and policy.

**Placing tool text at P5 or adding a "Tool output" label only reduces the chance of confusion. It cannot replace the gate before tool execution.**

## 3. The trust scope of Memory

- In the reference repositories
  - Chat Gun's `context-integration.ts` builds recall conditions with Principal, Tenant, and Scope only when it has a canonical execution context and a Memory provider.
  - `memory-governance-service.ts` also checks whether the namespace matches the trusted identity, and applies read authorization to candidate records.

These are identifiable data-scope controls. Memory content itself remains historical data, and passing read authorization does not give it the authority to modify the current instruction.

## 4. The design judgment of this article

**Each Context block keeps its source and purpose, data conflicts are resolved according to the current task and product policy, and code gates decide external effects.**

- Claiming that "an entry point blocks Prompt Injection" requires an end-to-end counterexample test from that entry point to tool execution.
- The current source classification and unit tests are not sufficient to support that conclusion.

## Implementation references and scope

The [chat-gun](https://github.com/HsienW/chat-gun) discussion is limited to the code integration points checked for this article. It does not imply that any entry point has passed end-to-end Prompt Injection verification.

| Reference | Implementation points to inspect |
|---|---|
| Sources and priority | `backend/src/context/context-source.ts` |
| Source reference record | `backend/src/context/context-manifest.ts` |
| Memory recall conditions | `backend/src/agents/context-integration.ts` |
| Memory read authorization | `backend/src/memory/governance/memory-governance-service.ts` |

For the design boundary, see [Agent Harness design principles](../foundations/04-design-principles.md).

## Further reading

- [Memory and compaction](./04-memory-and-compaction.md): trace when historical data returns to the current Context.
- [Context assembly](./02-context-assembly.md)
- [Input and Context overview](./README.md)
