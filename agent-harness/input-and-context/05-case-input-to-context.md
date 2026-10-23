# Input to Context: one request end to end

[English](./05-case-input-to-context.md) | [繁體中文](./05-case-input-to-context-zh-TW.md)

> Suppose the user once said "please keep answers short," and now tells the Chat Gun chatbot: "Explain the limitations of this design in full, and do not leave out details this time."

**Before this sentence reaches the model, it passes through three integration points: input governance, Graph messages, and Context assembly. Each point can prove something different.**

This article uses the scenario to place [input normalization](./01-input-normalization.md), [Context assembly](./02-context-assembly.md), [the trust boundary](./03-context-trust-boundary.md), and [Memory](./04-memory-and-compaction.md) on one path.

**This article checks whether one request can be traced to the material the model actually sees, with each omission explained.**

- This is not a production request that has been executed.
- The following sections describe the integration points the code passes through and the data to check. They do not claim to have captured that run's model Prompt or manifest.

## 1. Entry: recognize a new Prompt first

> `chatbotGraph` is wrapped by `applyInteractionGovernance`.

- `beforeRun` in `interaction-runtime.ts` calls `readNormalizedAgentInput`, which turns valid text into a `prompt` input, or rejects blank input, malformed input, and conflicting `kind` values before dispatch.
- The raw input also produces a digest reference.
- If the entry point receives a `command`, the current reader returns `unsupported_command`, and the request cannot be described as a model task from that point on.

### Where two fields meet

**Interaction governance stores `normalizedInput`, but the chatbot's Context assembly takes the latest `HumanMessage` from the Graph's `state.messages`.**

For the scenario to hold, check that the message the entry point sends into the Graph actually contains the current request and matches the normalized text. Similar field names do not mean the code guarantees that the two match word for word.

## 2. Assembly: separate the current request from past preferences

> If the chatbot's Context feature flag is enabled, `chatbot.ts` calls `assembleAgentContext`.

- System Policy is P0.
- The latest user message becomes the P1 current task.
- Earlier conversation becomes P4 recent messages.
- "Please keep answers short" can appear separately as P3 Memory only when the system has a canonical execution context and a Memory provider, and successfully recalls the relevant record. Without a recall result, the case table cannot assume that it exists.

![Two paths in the scenario: interaction governance produces normalizedInput and rawInputRef; the chatbot takes the current task from state.messages; the two must be checked for consistency before Memory is added conditionally and the model input is assembled.](../../assets/agent-harness/input-and-context/05-case-input-to-context/05-two-input-paths-en.svg)

### Evidence to keep at each integration point

| Integration point | Evidence to check in this case | Cannot be inferred directly |
|---|---|---|
| Input governance | `kind`, normalized text, raw input reference | Attachments or commands reached the model |
| Graph messages | Latest `HumanMessage` and earlier messages | Always equal to the normalized text |
| Context assembly | Retained blocks, `truncatedBlocks`, manifest | Every piece of Memory was selected |
| Model call | Assembled text passed to `llm.invoke` | The reply follows the current request |

### The current request should lead the answer

When the budget is sufficient, the model may see both the current request for a detailed explanation and the past preference for short answers.

**Correct task judgment lets the current explicit request lead the answer, with the past preference serving only as background. The P1 and P3 ordering keeps the former first when the budget runs short, but it does not prove that the model follows the rule.**

### Handle the two failures separately

- When the Context exceeds the hard limit, `chatbot.ts` turns `ContextHardLimitError` into an error reply.
- When Memory recall fails, the assembler can continue with the current task and record the reason in the manifest.

## 3. Turning the scenario into a verifiable record

> The next step is to fix one request, its Graph messages, the feature flag, the Memory provider configuration, and the model's capacity limits, then capture `blocks`, `truncatedBlocks`, and `manifest` from `AssembledAgentContext` along with the text actually passed to the model.

- The manifest keeps only source references and budget information, and references other than Memory are hashed; it is not enough on its own to reconstruct the full Prompt.
- An assembly result from a unit test proves at most how the assembler behaves with that fixture. It cannot support a claim that the production chatbot entry point produced the same content.

## 4. This request stops at the Context boundary

If the model later proposes a tool action, whether it executes still belongs to [action governance](../foundations/04-design-principles.md).

**Context source ordering cannot replace authorization.**

## Implementation references and scope

The [chat-gun](https://github.com/HsienW/chat-gun) discussion is limited to the code integration points checked for this article. This article did not capture an assembly result from the production chatbot entry point.

| Reference | Implementation points to inspect |
|---|---|
| Interaction governance and pre-dispatch check | `backend/src/platform/interaction-runtime.ts` |
| Input reading and normalization | `backend/src/runtime/input/read-normalized-agent-input.ts` |
| Chatbot branch and error reply | `backend/src/agents/chatbot.ts` |
| Agent integration and Memory recall | `backend/src/agents/context-integration.ts` |
| Block assembly and manifest | `backend/src/context/agent-context-assembler.ts` |

## Further reading

- [Input normalization](./01-input-normalization.md)
- [Context assembly](./02-context-assembly.md)
- [Context trust boundary](./03-context-trust-boundary.md)
- [Memory and compaction](./04-memory-and-compaction.md)
- [Agent Harness design principles](../foundations/04-design-principles.md)
