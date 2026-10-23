# Interaction and Feedback

[English](./README.md) | [繁體中文](./README-zh-TW.md)

This section follows [Action Governance](../action-governance/README.md). It covers how the Harness handles new input during an active task, controls visible Run ownership, resumes clarification, makes cancellation decisions, and turns inspected failures into regression material.

Six article drafts are available in both English and Traditional Chinese, with eight diagrams supplied as separate SVG and HTML files in each language. The article links below open the English drafts; the final column lists their filenames. The [Traditional Chinese guide](./README-zh-TW.md) retains the article topics, verification boundaries, diagram candidates, and pinned source index.

## Reading order

| Number | Article | Question | English filename |
|---|---|---|---|
| 01 | [In-flight input and interaction policy](./01-in-flight-input-policy.md) | Is new input a clarification answer, intent revision, cancellation, duplicate, or independent task? | `01-in-flight-input-policy.md` |
| 02 | [Run ownership and visible state](./02-run-ownership-and-visible-state.md) | Which execution may update the current view, and when must stale cleanup do nothing? | `02-run-ownership-and-visible-state.md` |
| 03 | [Clarification and task resumption](./03-clarification-and-resume.md) | Which wait receives an answer, and what must be checked before resuming? | `03-clarification-and-resume.md` |
| 04 | [Cancellation, correction, and committed effects](./04-cancellation-and-correction.md) | What can still be stopped, and which effects require separate handling? | `04-cancellation-and-correction.md` |
| 05 | [Feedback, evaluation, and regression cases](./05-feedback-and-evaluation.md) | How does a failure become reproducible, comparable evaluation material with appropriate data handling? | `05-feedback-and-evaluation.md` |
| 06 | [Control flow from a follow-up to feedback](./06-case-interaction-to-feedback.md) | Can input ownership, resumption, stale output, and regression evidence be traced through one scenario? | `06-case-interaction-to-feedback.md` |

## Scope and implementation boundaries

- Input and Context establishes normalized task material. This section decides how that input relates to an active task.
- Behavior and Capabilities covers effective instructions and capability selection. This section follows interaction decisions during execution.
- Action Governance covers tool approval and side effects. Clarification answers have their own wait and response contract; cancellation decisions reuse effect-state evidence.
- Agent Runtime will cover execution, checkpoints, and recovery mechanisms. This section does not introduce another general Run queue or worker runtime.

## Implementation findings

- An unconfigured `INTERACTION_POLICY` makes the wrapper a no-op. Its policy type defaults do not establish enabled production governance.
- The inspected `enqueue` consumer rejects local dispatch with `NATIVE_QUEUE_OWNERSHIP_REQUIRED`; it does not provide a local durable queue.
- Semantic classification helpers can produce tentative results. The current orchestrator stops dispatch when classification confirmation is required; a complete confirmation UI and return path remain to be checked.
- Clarification can preserve the Task while creating a new Run and generation. Correlation, expiry, response Schema, manifest compatibility, and single consumption require distinct checks.
- Cancellation needs actual dependency injection and downstream consumers. The production orchestrator constructor inspected here supplies no default `decideCancellation`; superseding ownership does not itself prove the old execution stopped.
- The stream wrapper emits interaction events before forwarding chunks. UI activity filtering depends on supplied generation metadata; the drafts now trace App and BFF wiring while retaining limits on message and terminal-callback coverage.
- Feedback helpers and a workflow CLI prepare dataset cases and compare supplied experiment snapshots. They do not automatically launch experiments, modify Prompts, retrain models, or deploy changes.
- Input allowlists and trace ID digests do not establish complete PII removal. Review allowed field values and expected outputs separately.

## Sources and verification status

GUN-ATLAS v1.0 supplied repository locations. The requested session file could not be found at either the supplied path or the corrected `.codex` path and is not recorded as inspected evidence. The attachment's interview-answer instructions do not define this section's format.

| Source | Pinned commit | Use |
|---|---|---|
| [Chat Gun](https://github.com/HsienW/chat-gun/tree/92a5673003e4ac4ec09be8b1b350f42dbf4a5e69) | `92a5673003e4ac4ec09be8b1b350f42dbf4a5e69` | Interaction orchestration, ownership, classification, clarification, cancellation, UI state, and feedback evaluation. |
| [Claude Code Best](https://github.com/claude-code-best/claude-code/tree/77a7934e15d69da13879112ed7db695c9ee7a52a) | `77a7934e15d69da13879112ed7db695c9ee7a52a` | Third-party comparisons for QueryGuard, prompt submission, queuing, and query abort. |

CCB is a third-party repository, not evidence of mandatory behavior in the current official Anthropic product. Source, test cases, and a historical runtime-boundary document were inspected. No unit, integration, E2E, live Provider, external MCP, database concurrency, or deployment failure tests were run during this work.

The final article uses a teaching scenario supported by separate wrapper, guard, clarification, UI, and feedback tests. Those tests must not be described as one executed production E2E flow. Eight diagrams are available under `assets/agent-harness/interaction-and-feedback/`, with editable SVG/HTML and separate English variants linked from the articles.

## Findings from drafting

The articles additionally inspect Deep Researcher interrupt persistence and resumption, App event handling, BFF proxying, and the default experiment runner. They document a contract mismatch: the UI candidate payload includes `candidateIndex`, while the default strict clarification response Schema does not accept that field. Separate UI and mock-wrapper tests do not establish a successful integrated resume.

The App both stops the transport and submits business cancellation. Actual cancellation still requires the missing default callback wiring and downstream consumers. Evaluation coverage also distinguishes Agent timeout from metric execution, reported costs from a complete spending cap, and observed tool-call data from external effect evidence. No source-repository tests or live experiments were run during drafting.
