# Agent Harness foundations

[English](./README.md) | [繁體中文](./README-zh-TW.md)

These five articles first define what the Harness is responsible for, then examine how decisions enter execution, and finally check the full path through a file-editing task. Read them in order: each article develops the question left by the preceding one into a concrete flow.

## Reading order

1. [What is an Agent Harness in practice?](./01-what-is-agent-harness.md): Define the Harness's responsibilities between the Agent Product and Runtime. Later articles use the terms established here.
2. [Harness control surfaces](./02-control-surfaces.md): Break the definition into five inspectable decision surfaces: input, Context, behavior and capabilities, action governance, and interaction and feedback.
3. [The Harness/Runtime boundary](./03-harness-runtime-boundary.md): Trace the execution conditions the Harness supplies, when the Runtime returns for decisions, and who owns state and failure.
4. [Agent Harness design principles](./04-design-principles.md): Draw engineering tradeoffs from those boundaries, including selecting Context for the task, distinguishing instruction authority from data authority, checking actions, and verifying rules through production entry points.
5. [A complete task walkthrough](./05-one-turn-walkthrough.md): Connect input, Context, tool results, authorization, failure, and completion evidence through a file-editing scenario, checking how the responsibilities in the first four articles work together.
6. Continue with [Input and Context](../input-and-context/README.md), which expands on input normalization, model-visible material, source authority, and Memory.

- For the implementation references used in these articles, distinguish observed code paths, capabilities integrated into entry points, available modules, and design proposals. A teaching scenario is not a record of a production run.

  - [claude-code-best](https://github.com/claude-code-best/claude-code)
  - [chat-gun](https://github.com/HsienW/chat-gun)
