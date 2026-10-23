# Input and Context

[English](./README.md) | [繁體中文](./README-zh-TW.md)

This section follows the [foundations](../foundations/README.md): it traces how the Harness receives a task, decides which material the model may see, and preserves the boundary between instructions and data. Read the five articles in order. Each one distinguishes the code paths that have been read, the capabilities integrated into a production entry point, and the author's design judgment, and marks the claims that still need execution evidence.

## Reading order

| Article | Question and boundary | Main references |
|---|---|---|
| [01. Input normalization](./01-input-normalization.md) | What becomes a task input before model execution? Stops at the normalized result and its source. | claude-code-best `processUserInput`; Chat Gun `normalized-agent-input.ts`, `read-normalized-agent-input.ts`, and `execution-composition-root.ts`. |
| [02. Context assembly](./02-context-assembly.md) | Which blocks enter a model call, and how are order and budget decided? | claude-code-best query preparation; Chat Gun `context-integration.ts`, `context-source.ts`, and `agent-context-assembler.ts`. |
| [03. Context trust boundary](./03-context-trust-boundary.md) | Which sources can instruct the system, and which remain task data? | Chat Gun source classification and current-task priority; the instruction and tool-output boundaries described in the foundations. |
| [04. Memory and compaction](./04-memory-and-compaction.md) | How do recall and compression keep a task going without overriding the current request? | Chat Gun Memory recall, budget, compression, and manifest; claude-code-best's separate compacting paths. |
| [05. Input to Context](./05-case-input-to-context.md) | Can one request be traced to the content the model actually sees? | The Chat Gun chatbot entry point, with each inclusion, exclusion, and failure point identified. |

The articles are based on the code paths that can currently be read. Claiming end-to-end Memory behavior for a deployment in Article 04 still requires checking the provider configuration and the relevant Agent entry points. The scenario in Article 05 needs a reproducible request and its Context manifest before it can become a record of an actual run. Budget priority only explains tradeoffs; it cannot by itself support a claim that Prompt Injection is blocked.

## Evidence convention

Name the repository and path beside each implementation claim. Use **observed** for a code path that has been read, **integrated** only when an actual entry point calls it, and **design judgment** for a rule the author proposes. Recheck Chat Gun's current branch before treating a historical conversation or a foundation article as evidence of its present behavior. The earlier Codex session provides Chat Gun repository context; the claude-code-best path references come from the foundations and are checked again in each article.
