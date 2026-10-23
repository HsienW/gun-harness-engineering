# Context assembly

[English](./02-context-assembly.md) | [繁體中文](./02-context-assembly-zh-TW.md)

> One task does not call the model only once. When the user adds a requirement, a tool returns data, or a long conversation fills the window, the model may see different material on the next call.

**The Harness therefore needs to decide the source, size, and inclusion of material before each call, and keep a record that can explain those tradeoffs. This decision sits next to, but is separate from, deciding which tool may modify a resource.**

[The previous article](./01-input-normalization.md) explains how entry-point material becomes a task input the system can inspect.

**This article follows what happens before one model call: which sources are selected, ordered, shortened, or excluded.**

- This article uses Chat Gun's readable code path to explain Context details; claude-code-best serves only as a comparison of entry-point structure.
- Whether a tool may execute belongs to action governance and is outside the scope of this article.

## 1. From messages to source blocks

> Chat Gun's `chatbot.ts` calls `assembleAgentContext` when `CONTEXT_ASSEMBLY_ENABLED_CHATBOT` is enabled, and uses `legacyChatbotPrompt` when it is disabled.

- The code enables the feature flag by default, but whether a deployment actually uses the new path still depends on its environment configuration.
- `assembleAgentContext` takes the latest `HumanMessage` in `state.messages` as `current_task`, and lists the other messages as `recent_messages` or `tool_output` according to their type.
- Callers can also supply `active_state` and tool output separately.

**This step reads Graph message state. It cannot be claimed to equal the `normalizedInput.text` produced in [the previous article](./01-input-normalization.md).**

### Sources and budget priority

`context-source.ts` maps sources to budget priorities:

| Priority | Source | Main purpose |
|---|---|---|
| P0 | `system_policy` | Product policy |
| P1 | `current_task` | Current request |
| P2 | `active_state` | State of the running execution |
| P3 | `memory` | Historical memory within scope |
| P4 | `recent_messages` | Recent conversation |
| P5 | `tool_output` | Material returned by tools |

- This table describes the order of tradeoffs when the budget runs short. It is not a complete table of instruction authority.
- Labeling tool results as P5 does not guarantee that their text will not influence the model; [the trust boundary article](./03-context-trust-boundary.md) addresses that question.

## 2. What happens when the budget runs short

> `buildAgentContext` first turns sources into blocks and estimates their Tokens, then combines the product budget, the model's Context Window, and the output reserve to calculate the effective limit.

![Context budget decisions: candidate blocks P0 to P5 are shortened or removed according to the effective limit; the assembler raises an error when P0, or P0 plus P1, exceeds the limit, and on success produces model-visible text, a manifest, and truncatedBlocks.](../../assets/agent-harness/input-and-context/02-context-assembly/02-context-budget-en.svg)

### Order of shortening and removal

1. The default compression strategy first shortens P5 tool results
2. It then shortens the P4 conversation
3. If the material still does not fit, it removes these lower-priority blocks
4. The assembler can then remove whole classes of blocks in order from P5 to P2

### Hard limit

**If P0 alone exceeds the limit, or P0 plus P1 still does not fit, the assembler throws `ContextHardLimitError` instead of silently dropping the policy or the current task.**

- `chatbot.ts` turns this error into a reply message, and the normal model call does not continue on that branch.

## 3. The assembly result and the manifest

> After a successful assembly, the model receives text formatted from the retained blocks.

- `ContextManifest` records the kind, reference, estimated Tokens, effective limit, compression actions, and reason codes for retained sources; references other than Memory are hashed first.
- The assembly result also exposes `truncatedBlocks` for inspection, while the manifest itself does not list the original text of each removed block.

**Reconstructing one set of tradeoffs requires keeping the request, the assembly result, and the corresponding execution correlation together. A single manifest is not enough to claim knowledge of everything the model saw.**

## 4. Comparing claude-code-best's entry points

[The foundations](../foundations/03-harness-runtime-boundary.md) show that claude-code-best's Interactive and Headless/SDK entry points each have their own outer preparation, but both pass the assembled conditions to the core `query()`.

- The data structures and compression mechanisms of the two products cannot be treated as the same.

## Implementation references and scope

The [chat-gun](https://github.com/HsienW/chat-gun) discussion is limited to the code integration points checked for this article. Each Agent's feature flag and actual call site still require evidence from the corresponding environment.

| Reference | Implementation points to inspect |
|---|---|
| Chatbot branch and error reply | `backend/src/agents/chatbot.ts` |
| Agent integration | `backend/src/agents/context-integration.ts` |
| Sources and priority | `backend/src/context/context-source.ts` |
| Block assembly and removal | `backend/src/context/agent-context-assembler.ts` |
| Effective limit | `backend/src/context/context-limit.ts` |
| Tradeoff record | `backend/src/context/context-manifest.ts` |

The [claude-code-best](https://github.com/claude-code-best/claude-code) discussion reuses the entry-point references from [the foundations](../foundations/03-harness-runtime-boundary.md). It does not represent the complete behavior of the current official version.

## Further reading

- [Context trust boundary](./03-context-trust-boundary.md): separate budget priority from instruction authority.
- [Input normalization](./01-input-normalization.md)
- [Input and Context overview](./README.md)
