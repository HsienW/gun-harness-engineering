# Agent Harness foundations

[English](./README.md) | [繁體中文](./README-zh-TW.md)

These five articles establish what the Harness controls, where its responsibility ends, and how those decisions appear during one task. Read them in order: each article gives the next one a more concrete question to answer.

## Reading order

1. [What is an Agent Harness?](./01-what-is-agent-harness.md) defines the Harness as a product responsibility and places it alongside the Agent Product and Runtime. Start here for the terms used throughout this section.
2. [Harness control surfaces](./02-control-surfaces.md) identifies the decisions over input, Context, behavior and capabilities, action governance, and interaction. It turns the definition into places a system can inspect and enforce.
3. [The Harness/Runtime boundary](./03-harness-runtime-boundary.md) follows those decisions into execution: what the Harness passes to the Runtime, when the Runtime asks for another decision, and who owns state and failure.
4. [Agent Harness design principles](./04-design-principles.md) draws engineering judgments from those boundaries, including task-scoped Context, the authority of data sources, action checks, and verification through real entry points.
5. [A complete task walkthrough](./05-one-turn-walkthrough.md) traces a file edit from user request to final reply, including Context assembly, tool results, authorization, failure, and evidence for completion.

The Claude Code paths in these articles are implementation references. Chat Gun examples distinguish an observed code path from a capability integrated into an entry point, an available module, or a design proposal. An illustrative task is not a record of a production run.

Continue with [Input and Context](../input-and-context/README.md), which covers input normalization, model-visible material, source authority, and Memory.
