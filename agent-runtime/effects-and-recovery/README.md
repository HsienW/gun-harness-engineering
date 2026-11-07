# Effects and Recovery

[English](./README.md) | [繁體中文](./README-zh-TW.md) | [简体中文](./README-zh-CN.md)

[Back to Runtime overview](../README.md) · [Foundations guide](../foundations/README.md) · [Model and Tool Execution](../model-and-tool-execution/README.md)

> In the following content:
> CCB is short for [claude-code-best](https://github.com/claude-code-best/claude-code).
> Chat Gun refers to [chat-gun](https://github.com/HsienW/chat-gun).

An external service completes a write, but its response is lost before reaching the Runtime. After restart, the Checkpoint still precedes tool execution and the user submits the same request again. Graph recovery, request redelivery, and external-operation repetition each have different conditions. The Runtime must first recover execution correlation and assess effect evidence before choosing result reuse, another attempt, continued waiting, or manual handling.

Drafts of all six Chinese and English articles are complete. They build on lifecycle and tool-call boundaries from the preceding two categories, expanding recovery state, replay identity, effect ledgers, reconciliation, and compensation. Article plans, diagram candidates, and source scope follow. Round one completed four bilingual SVG/HTML pairs; round two completed four English diagrams. English articles reference all eight English diagrams; round-two Chinese diagrams remain unproduced. English filenames retain the stem without `-zh-TW`, and article language links switch editions.

## Reading order

| Part | Article | Question to answer | Filename |
|---|---|---|---|
| 01 | [Checkpoints and recovery state](./01-checkpoints-and-recovery-state.md) · [繁體中文](./01-checkpoints-and-recovery-state-zh-TW.md) | Which state, versions, and message evidence locate a safe continuation point after restart? | `01-checkpoints-and-recovery-state.md` |
| 02 | [Idempotency and replay identity](./02-idempotency-and-replay-identity.md) · [繁體中文](./02-idempotency-and-replay-identity-zh-TW.md) | How are duplicate requests, the same Tool Call, and the same business effect distinguished, and when may results be reused? | `02-idempotency-and-replay-identity.md` |
| 03 | [Effect ledgers and commit boundaries](./03-effect-ledger-and-commit-boundaries.md) · [繁體中文](./03-effect-ledger-and-commit-boundaries-zh-TW.md) | What records surround dispatch, and what failure windows separate external commit from local persistence? | `03-effect-ledger-and-commit-boundaries.md` |
| 04 | [Effect reconciliation and retry decisions](./04-reconciliation-and-safe-retry.md) · [繁體中文](./04-reconciliation-and-safe-retry-zh-TW.md) | How are ambiguous results verified, and which evidence and limits permit redispatch? | `04-reconciliation-and-safe-retry.md` |
| 05 | [Compensation and manual recovery](./05-compensation-and-manual-recovery.md) · [繁體中文](./05-compensation-and-manual-recovery-zh-TW.md) | How does compensation handle partially completed operations, and what remains after failed or irreversible compensation? | `05-compensation-and-manual-recovery.md` |
| 06 | [Recovering an ambiguous commit](./06-case-ambiguous-commit-recovery.md) · [繁體中文](./06-case-ambiguous-commit-recovery-zh-TW.md) | Can evidence support each decision from external commit through response loss, restart, and subsequent handling? | `06-case-ambiguous-commit-recovery.md` |

## Architecture and responsibilities

The diagram is a reading model whose arrows mark handoffs to inspect. It prescribes no shared API for the two sources and does not imply that one function performs all recovery work.

```mermaid
flowchart TD
    Load[01 Read Checkpoint / Manifest / durable evidence] --> Valid[Validate versions, messages, and execution position]
    Valid --> Identity[02 Validate request, replay, and business-effect identity]
    Identity --> Ledger[03 Read Ledger and result references]
    Ledger --> Evidence[04 Reconcile external effects]
    Evidence --> Reuse[Committed with an available result: reuse]
    Evidence --> Retry[Proven non-commit and retry allowed: recheck then dispatch]
    Evidence --> Park[Insufficient evidence: wait or request manual intervention]
    Reuse --> Continue[Check current Run and governance before continuation]
    Partial[Partial business completion with compensation eligibility] --> Compensate[05 Execute registered compensation actions]
    Compensate --> Record[Save compensation results and unresolved items]
    Case[06 Failure case] -. Verify each stage .-> Evidence
```

### Division of work with other sections

| Section | What this section takes on | What existing sections retain |
|---|---|---|
| Foundations | Expand concrete Checkpoint, effect-evidence, and recovery data and failure windows. | Runtime terms, basic states, and design principles. |
| Execution Lifecycle | Read Run/Task/Step, waiting, and terminal evidence to decide available recovery actions. | Admission, ownership, supersede, cancellation requests, and Terminal commits. |
| Model and Tool Execution | Continue from a started or returned runner through Ledger, result references, and effect interpretation. | Model-stream assembly, tool scheduling, capacity, and result envelopes. |
| Harness Action Governance | Consume trusted authorization, confirmation, and operation correlations; redispatch still follows governance conditions. | Policy definition, Resource scope, and human-approval contracts. |
| Events and Operations | Establish evidence requirements for recovery classification, reconciliation, and compensation. | Event delivery, Trace, alerts, operational interfaces, and deployment drills. |

### Data to keep separate

| Data | Supported judgment | What still needs other evidence |
|---|---|---|
| Graph Checkpoint | Messages, node position, and pending work. | External-operation commit and readability after deployment. |
| Interrupt/Execution Manifest | Waiting identity, reply contract, expiry, and version compatibility. | Successful execution handoff after Resume consumption. |
| Task/Step/Ledger | Local lifecycle, execution attempts, and persisted effect states. | External commit truth after a lost response. |
| Result Reference | Retrieve an existing result when scope, version, and policy permit. | Whether a lost result permits repeating the original operation. |
| External Operation evidence | Committed, provably not committed, or still unknown. | Whether retry is currently allowed or compensation is required. |

## 01. Checkpoints and recovery state

### Article scope

Start with data reads after a process stops, relating Checkpoint, Interrupt Manifest, Execution Manifest, and durable execution evidence. Retain Execution Lifecycle's Resume consumption rules and focus on whether recovery data is usable and how inconsistent data exits.

Article topics:

1. Separate persistence purposes of Graph state, Session transcripts, Task/Step, and external-effect evidence.
2. Cross-check `threadId`, `runId`, `taskId`, `interruptId`, and scope; Thread alone cannot establish recovery responsibility.
3. Graph configuration hashes, Schema versions, and Manifest compatibility; `migratable` still requires an actual migration implementation.
4. Sanitization and parking conditions for unpaired Tool Calls/Results, duplicate results, incomplete arguments, and partial Assistant Messages.
5. Classification priority among `not_started`, `executing`, `committed`, `unknown`, `waiting_user`, and `terminal`.
6. Failure windows among Resume consumption, `updateState()`, and `Command({ resume })`, plus Checkpointer durability requirements.

### Located sources and verification requirements

Chat Gun `conversation-recovery.ts` finds the Manifest, validates correlation, then reads the Checkpoint, sanitizes history, and loads durable evidence. `langgraph-checkpoint-adapter.ts` separates history updates from Graph invoke into two calls; the inspected order does not establish a shared transaction with Resume consumption.

`last-execution-point.ts` checks Terminal before Ledger committed/unknown and waiting. The classifier accepts input without a Checkpoint, while `recover()` returns manual intervention first when the Checkpoint is missing. Separate module capability from paths reachable through this consumer. `execution-manifest.ts` distinguishes compatible, migratable, and incompatible; classification alone does not establish an automatic migrator.

CCB `QueryEngine.ts` provides a transcript-record and message-chain comparison. The article additionally inspects transcript recording/loading helpers in `sessionStorage.ts` and recovery-data assembly in `sessionRestore.ts`, retaining disabled-configuration and asynchronous-write conditions. Preserve the Deep Researcher `MemorySaver` usage in Chat Gun; an Adapter interface cannot establish cross-process persistent Checkpointers for every Graph.

Diagram candidate: recovery reads across Checkpoint, Manifest, Ledger, and external evidence, plus failure windows between Resume consumption and Graph handoff.

## 02. Idempotency and replay identity

### Article scope

Separate user resubmission, Runtime replay, physical Attempts, and business effects to explain each identifier's scope. Derive reuse and conflict conditions from their relationships; TTL, hashes, or locks alone cannot establish external-effect idempotency.

Article topics:

1. Uses of Request Dedup Key, Replay Key, Tool Execution, Attempt ID, and Business Effect Key.
2. Conflict handling for the same Replay Key with a different `requestHash`.
3. Correlating a new Run, different Tool Call IDs, and the same business operation; derive business keys from descriptors and trusted scope.
4. Local unique constraints and external-service idempotency contracts, including their separate retention periods.
5. TTL expiry, failed-record reacquisition, and old calls still executing; subsequent owner/fencing requirements.
6. Rejecting committed-result reuse when scope, tool version, cache state, or payload is invalid.

### Located sources and verification requirements

The Chat Gun runner looks up the Replay Key before comparing `requestHash`. Existing non-committed records require reconciliation rather than unconditional reexecution. Ledger business-effect uniqueness includes tenant, scope, and business key; external Operation namespace/ID has a separate unique constraint.

`identity.ts` and `idempotency-key.ts` now delegate to `@gun-ai/harness-kernel`/contracts. The article inspects the installed `0.1.0-alpha.1` package and compares Replay Key construction in `src/side-effect.ts` with exported `dist/side-effect.js`; key fields follow that implementation. `PgIdempotencyGuard` failed/expired reclamation and the runner's Ledger are different mechanisms with distinct safety guarantees.

`result-reference-store.ts` checks scope, tool version, cache state, and policy during resolve. When retrieval fails, retain the possibility that the effect committed but its result is unavailable; do not turn that state directly into redispatch.

Diagram candidate: identity relationships among requests, logical calls, physical Attempts, and business effects, plus the result-reuse validation path.

## 03. Effect ledgers and commit boundaries

### Article scope

Follow a tool operation with a Side-effect Descriptor through durable prepare, claim, Attempt, external I/O, Result Reference, and Ledger commit. List records remaining after a process stops in each window, giving reconciliation a concrete starting point.

Article topics:

1. Record and state differences among Tool Execution, Business Effect, and Compensation Execution.
2. Prepare exits: claimed, existing_committed, not_claimed, conflict, and unavailable.
3. Obtain execution eligibility and record the Attempt before external I/O dispatch.
4. Separate transaction boundaries for external commit, Result Reference persistence, Ledger commit, and Task Step persistence.
5. Interpreting external success with failed persistence, result-validation failure, late cancellation, and response loss.
6. Stop dispatch when the Ledger is unavailable or state conditions fail; retain unknown evidence after restart.

### Located sources and verification requirements

`PgBusinessEffectLedger.prepare()` uses a database transaction for local claim, while state transitions use expected status/state. `commitExecutionAndBusinessEffect()` can update two local records transactionally; external-service I/O and Result Reference persistence do not thereby join that transaction.

`persistCommittedResult()` saves the Result Reference before committing the Ledger. Even when output validation fails, the code may retain committed-effect records and return deferred. Compare `SIDE_EFFECT_OUTPUT_VALIDATION_FAILED` and `SIDE_EFFECT_PERSISTENCE_UNCERTAIN`, separating format repair, result recovery, and external-effect repetition.

`pipeline.ts` can construct the runner, Ledger, and compensation objects from a Pool. Verify each production tool entry's Pipeline enablement, descriptor provision, and adapter external contract. CCB tombstone/discard provides a comparison for message/result cleanup, not proof of external-I/O reversal.

Diagram candidate: failure timing from external Commit through local Ledger/Task Step persistence, marking actual transaction scopes.

## 04. Effect reconciliation and retry decisions

### Article scope

Turn `committed`, `not_committed`, and `unknown` evidence into explicit next actions, then follow actual redispatch. Result reuse, retry eligibility, and continued waiting are separate decisions with distinct reasons and current limits.

Article topics:

1. Query effects using trusted scope, Business Effect Key, and available external Operation correlations.
2. Record and reuse committed effects with available results; stop blind repetition when committed results are unavailable.
3. Evidence quality for `not_committed`, external-query lag, and consistency requirements.
4. Retryable classification, Retry Budget, Backoff, Retry-After, cancellation, and pre-dispatch authorization checks.
5. Exits for missing Reconcilers, query exceptions, unknown results, and failed local state transitions.
6. Separate recovery-service commit/retry decisions from consumers that execute them.

### Located sources and verification requirements

`decideReconciliationAction()` chooses commit for committed, retry only for not_committed with `canRetry`, and defer otherwise. The runner retry branch also requires successful Tool Execution and Business Effect transitions. Committed queries without results return `SIDE_EFFECT_COMMITTED_RESULT_UNAVAILABLE`.

Conversation Recovery reconciled results can carry retry decisions, but that branch does not directly call the tool runner. The runner separately defers replay of existing non-committed records. Do not combine those entry points into unconditional automatic recovery. Inspect `routeRecoveryContinuation()` executing routes alongside evidence trust, actual takeover points, and dispatch consumers.

Further inspection of production MCP descriptors finds a Reconciler returning unknown with `MCP_RECONCILIATION_QUERY_UNAVAILABLE`. Other tools and services require individual verification; a generic interface cannot establish end-to-end safe retry. Whether a cross-service not-found response proves non-commit belongs in adapter contracts and fault tests.

Diagram candidate: three reconciliation states leading to reuse, retry, or manual intervention, with cancellation, budget, and state-write limits marked separately.

## 05. Compensation and manual recovery

### Article scope

Use partially completed business actions followed by failure to organize compensation targets, execution order, and compensation's own effect evidence. Describe what compensation changes and which states remain unrecoverable.

Article topics:

1. After establishing the original effect, select compensation, retention, or manual handling under business rules.
2. Eligibility based on Task state, successful Steps, and registered Compensation Actions.
3. Reverse Step/Action order, irreversible operations, and unregistered actions.
4. Compensation's own Execution, authorization requirements, idempotency, cancellation, and retry contract.
5. Executed compensation with failed persistence, plus duplicate risks on reentry.
6. Distinctions among original Task state, Compensation Result, uncompensated items, and manual-handling evidence.

### Located sources and verification requirements

`SagaOrchestratorImpl.compensate()` accepts partially_failed or cancelled Tasks, selects eligible succeeded Steps, and processes Steps and Actions in reverse. It skips irreversible Actions and requires manual intervention. The Registry default can be empty; inspect whether concrete actions are registered.

With committed execution and a Ledger, the code prepares/transitions Compensation Execution before executing actions. Successful compensation with failed persistence can yield `COMPENSATION_PERSISTENCE_UNCERTAIN`. After compensation, the Task becomes failed while `overallStatus` separately describes all_compensated or manual_intervention_required. Retain these states individually rather than calling all of them recovery success.

Inspection in this round does not establish cross-Worker single-execution guarantees, shared transactions, or full retry deduplication for compensation. Validate reentry, actions executed before record failure, missing committed references, and whether unknown effects are suitable compensation targets.

Diagram candidate: dual-Ledger timing for original and compensating operations, marking irreversibility, compensation failure, and uncertain persistence separately.

## 06. Recovering an ambiguous commit

### Article scope

A teaching case connects the first five articles through an external write that committed, lost its response, and encountered process restart. The main path assumes the external service supplies trusted commit evidence and results. Non-commit, continued uncertainty, compensation, and manual intervention remain independent branches.

Article topics:

1. List teaching correlations for Run, Step, Tool Call, Replay Key, Business Effect, and external Operation.
2. Original execution claims, dispatches the write, and loses the response after external commit; record the stopping point.
3. After restart, load durable state, validate Manifest/version, and sanitize usable history.
4. Reconcile committed effects with available results, repair local records, and recheck continuation eligibility.
5. Recheck retry conditions after proving non-commit; retain parking/manual branches for unknown effects or missing results.
6. When completed partial operations require compensation, switch to an independent compensation path and retain its results.

### Located sources and verification requirements

Build responsibility lanes from Chat Gun runner, Ledger, Conversation Recovery, and Saga modules. Mark which sequences one function already executes and which require external consumer wiring. Use CCB only to compare transcript/discard recovery scope; do not create a common execution flow spanning both repositories.

The validation matrix includes stopping before dispatch, response loss after external commit, Result Reference persistence failure, Ledger persistence failure, consumed Resume without Graph handoff, parameter conflicts for one Replay Key, and executed compensation with failed recording. Drill each separately rather than drawing them all as one normal execution.

Diagram candidate: complete failure lanes for Runtime, Checkpoint/Manifest, Ledger, external service, and recovery consumer.

## Diagram progress

Round one followed 03 → 04 → 05 → 06. All four diagrams provide Chinese and English versions; Chinese articles reference Chinese diagrams. English articles are complete, and English HTML links back to them. HTML retains interpretation, sources, teaching assumptions, and scope; SVG text is editable.

| Article | Diagram | HTML | SVG |
|---|---|---|---|
| 03 | Persistence order and failure windows across external commit, result references, Ledger, and Task Step | [Open Chinese HTML](../../assets/agent-runtime/effects-and-recovery/03-effect-ledger-and-commit-boundaries/03-external-commit-and-local-persistence.html) | [View Chinese SVG](../../assets/agent-runtime/effects-and-recovery/03-effect-ledger-and-commit-boundaries/03-external-commit-and-local-persistence.svg) · [English HTML](../../assets/agent-runtime/effects-and-recovery/03-effect-ledger-and-commit-boundaries/03-external-commit-and-local-persistence-en.html) · [English SVG](../../assets/agent-runtime/effects-and-recovery/03-effect-ledger-and-commit-boundaries/03-external-commit-and-local-persistence-en.svg) |
| 04 | Three reconciliation states: committed, not_committed, unknown and subsequent conditions | [Open Chinese HTML](../../assets/agent-runtime/effects-and-recovery/04-reconciliation-and-safe-retry/04-reconciliation-outcomes-and-retry.html) | [View Chinese SVG](../../assets/agent-runtime/effects-and-recovery/04-reconciliation-and-safe-retry/04-reconciliation-outcomes-and-retry.svg) · [English HTML](../../assets/agent-runtime/effects-and-recovery/04-reconciliation-and-safe-retry/04-reconciliation-outcomes-and-retry-en.html) · [English SVG](../../assets/agent-runtime/effects-and-recovery/04-reconciliation-and-safe-retry/04-reconciliation-outcomes-and-retry-en.svg) |
| 05 | Independent compensation after original commit; action success can still leave persistence uncertain | [Open Chinese HTML](../../assets/agent-runtime/effects-and-recovery/05-compensation-and-manual-recovery/05-compensation-and-persistence-uncertainty.html) | [View Chinese SVG](../../assets/agent-runtime/effects-and-recovery/05-compensation-and-manual-recovery/05-compensation-and-persistence-uncertainty.svg) · [English HTML](../../assets/agent-runtime/effects-and-recovery/05-compensation-and-manual-recovery/05-compensation-and-persistence-uncertainty-en.html) · [English SVG](../../assets/agent-runtime/effects-and-recovery/05-compensation-and-manual-recovery/05-compensation-and-persistence-uncertainty-en.svg) |
| 06 | Recovery lanes: lost response after external commit, effect checks, reconciliation, local repair, and result delivery | [Open Chinese HTML](../../assets/agent-runtime/effects-and-recovery/06-case-ambiguous-commit-recovery/06-ambiguous-commit-recovery-swimlane.html) | [View Chinese SVG](../../assets/agent-runtime/effects-and-recovery/06-case-ambiguous-commit-recovery/06-ambiguous-commit-recovery-swimlane.svg) · [English HTML](../../assets/agent-runtime/effects-and-recovery/06-case-ambiguous-commit-recovery/06-ambiguous-commit-recovery-swimlane-en.html) · [English SVG](../../assets/agent-runtime/effects-and-recovery/06-case-ambiguous-commit-recovery/06-ambiguous-commit-recovery-swimlane-en.svg) |

Geometry, links, and system-fallback font layout were checked. These checks do not replace Runtime, external-service, or database failure drills.

### Round-two English diagrams

These follow the Chinese articles; corresponding Chinese diagrams have not yet been created. English articles reference these diagrams, and HTML links back to the matching English articles.

| Article | Diagram | HTML | SVG |
|---|---|---|---|
| 01 | Recovery-evidence relationships | [English HTML](../../assets/agent-runtime/effects-and-recovery/01-checkpoints-and-recovery-state/01-recovery-evidence-map-en.html) | [English SVG](../../assets/agent-runtime/effects-and-recovery/01-checkpoints-and-recovery-state/01-recovery-evidence-map-en.svg) |
| 02 | Replay, Attempt, and business-effect identity | [English HTML](../../assets/agent-runtime/effects-and-recovery/02-idempotency-and-replay-identity/02-replay-attempt-and-effect-identity-en.html) | [English SVG](../../assets/agent-runtime/effects-and-recovery/02-idempotency-and-replay-identity/02-replay-attempt-and-effect-identity-en.svg) |
| 01 | Resume consumption and handoff windows | [English HTML](../../assets/agent-runtime/effects-and-recovery/01-checkpoints-and-recovery-state/01-resume-consume-and-handoff-windows-en.html) | [English SVG](../../assets/agent-runtime/effects-and-recovery/01-checkpoints-and-recovery-state/01-resume-consume-and-handoff-windows-en.svg) |
| 04 | Generation-conditional takeover race | [English HTML](../../assets/agent-runtime/effects-and-recovery/04-reconciliation-and-safe-retry/04-generation-conditional-takeover-en.html) | [English SVG](../../assets/agent-runtime/effects-and-recovery/04-reconciliation-and-safe-retry/04-generation-conditional-takeover-en.svg) |

## Source additions in this round

All six articles followed the attachment's code index into implementations, retaining reading order and adding these concrete handoffs.

| Article | Added source observations | Retained capability limits |
|---|---|---|
| 01 | CCB transcript leaf selection, parent/child chains, and new-message deduplication. | Conversation recovery does not prove external effects never committed. |
| 02 | Production MCP canonical-input business keys and Pipeline call indexes. | Stable argument representation does not automatically establish equivalent business meaning. |
| 03 | Registry source flags, executor factories, and outcome mappings. | Pipeline entry does not give every tool an effect Ledger. |
| 04 | Worker recovery classification, Reaper generation takeover, and runner-test assertions. | Decisions and ownership updates do not themselves requeue execution. |
| 05 | CCB file-history per-file restoration, deletion, and individual-error handling. | A rewind return does not prove every file was restored. |
| 06 | Stuck-Run takeover steps and two-contender failure scenarios. | Local generation updates do not provide remote fencing. |

Both specified session paths were checked again in this round, and the file remained unavailable. Additions follow the attachment index and local code inspection; the session conversation was not read, and source-repository tests or failure drills were not run.

## Sources and verification status

The attachment `GUN-ATLAS_v1_2026-10-12.md` provides Chat Gun and CCB indexes; local code was also read in this round. Memory-operation and interview guidance in the attachment is reference material, not article instructions. The specified session was found at neither `C:/Users/hsien.codex/sessions/...` nor `C:/Users/hsien/.codex/sessions/...`, so its conversation content is not cited.

Local Chat Gun sources are under `C:/D/ai-agent/chat-gun`, and CCB under `C:/D/ai-agent/claude-code-research/claude-code-main`. Git could not provide a Chat Gun revision in this round; the SHA256 table records the inspected content instead of automatically inheriting the preceding section's commit. Local wrappers now delegate to kernel packages; the articles additionally inspect identity and retry implementations in installed version `0.1.0-alpha.1`. CCB is third-party code, not Anthropic official source.

This round read implementations, some test cases, and database migrations without executing source tests, Postgres integration, process restarts, external Providers/services, compensation, or cross-Worker failure drills. Test filenames and case names identify later verification entry points and do not establish passes. The Ledger integration suite is also controlled by `SIDE_EFFECT_INTEGRATION_DATABASE_URL` and can skip when it is unset.

The articles separate source observations, actual entry wiring, teaching design, and unverified capabilities. Diagrams are planned under `assets/agent-runtime/effects-and-recovery/`, retaining Foundations' light background, blue focus, dashed regions, and rounded nodes with bilingual SVG/HTML. Round one completed bilingual diagrams for 03 commit windows, 04 reconciliation branches, 05 compensation persistence, and 06 recovery lanes. Round two completed four English diagrams from Chinese articles; Chinese diagrams remain pending.

### Inspected file fingerprints

Paths below are relative to the respective source repositories. SHA256 values identify local files inspected in this round and do not claim identical remote-branch content.


| Source | Relative path | SHA256 |
|---|---|---|
| Chat Gun | `backend/src/runtime/recovery/conversation-recovery.ts` | `f314b72240529a62674bc50a982395c59c0924670dd43456768732c1eaa48b35` |
| Chat Gun | `backend/src/runtime/recovery/langgraph-checkpoint-adapter.ts` | `a831cc4744d5b40848799cb80543b2001417ea25de6a0f944efa51cf499fdfb2` |
| Chat Gun | `backend/src/runtime/recovery/last-execution-point.ts` | `0b1d2444a91afd1121a0ab3f0dde5ab5c2baba06f8490d5a0a65a5293eee85f4` |
| Chat Gun | `backend/src/runtime/recovery/recovery-sanitizer.ts` | `f2f12b521e968b07fa251e98d75a401c240e63ac9fa95f769071042fb606f1d2` |
| Chat Gun | `backend/src/runtime/recovery/execution-manifest.ts` | `c6df220446197061412967579ca885f2aa532c92e43a35b8473bed3e669e5b03` |
| Chat Gun | `backend/src/runtime/side-effect/identity.ts` | `d59a5a90f310ea0c6999bf0074d07f2cb65b888027069af3a321ec09d85bb179` |
| Chat Gun | `backend/src/runtime/idempotency/idempotency-guard.ts` | `65d05a422a42aed3d7376c2841cb57c64908e2c6d0d9d8ea4ef47fd9b5985ef1` |
| Chat Gun | `backend/src/runtime/side-effect/business-effect-ledger.ts` | `77682fe69b54128a1dae4cffae58560a968a8d3c777a44e1791834d767b728b7` |
| Chat Gun | `backend/src/runtime/side-effect/tool-execution-runner.ts` | `88a438fea4af9b73ac5eb32f1436b0815ca387148532c674c84c0852e260997f` |
| Chat Gun | `backend/src/runtime/side-effect/side-effect-descriptor.ts` | `69bbdcaf98714457075962c4942d9a4c308a3e6e4a720f2a72e2eadb21d300ea` |
| Chat Gun | `backend/src/runtime/side-effect/result-reference-store.ts` | `de19124c484de83b5fa813e778385199db868e5400a2fded89f4d70a8c70939e` |
| Chat Gun | `backend/src/runtime/side-effect/reconciler.ts` | `be7348a443b4c6830a07843dfbf78f56a68dc5962fb012038b4727d72f9f2031` |
| Chat Gun | `backend/src/runtime/compensation/saga-orchestrator.ts` | `dbc9a3db228d74df3839f1f49b4865d28a63542ea2b55589b8214efff8edc678` |
| Chat Gun | `backend/src/runtime/compensation/compensation-registry.ts` | `729360d4eb72035b964179b3ebcd93b80d1bed3c88c926af6a3ac5dd67e5f690` |
| Chat Gun | `backend/src/runtime/tool-dispatch/pipeline.ts` | `4cb9795b0f0ac74a1fe22996da05d2b9ac8dc2b474f7d93076fe2f4135d1c358` |
| Chat Gun | `backend/src/runtime/persistence/migrations/006_create_business_effects.sql` | `c696c498d61577380186c86e191fe340227d56c0607dd4be709a65ef5bad1c9b` |
| CCB | `src/QueryEngine.ts` | `7df34a6a6d106927403d49e3405dfaf70da37cae5b644b658ebaf0b877988af6` |
| CCB | `src/services/tools/StreamingToolExecutor.ts` | `ee17dce704af73787c9ed76dbbc638167521e5c1f68c9436149378bb3358601c` |
| Chat Gun | `backend/src/runtime/recovery/interrupt-manifest-repository.ts` | `483d2ceb03955dad48d8e0588c698374983911cfef4f8821a484b93ccb7b6811` |
| Chat Gun | `backend/src/runtime/compensation/compensation-action.ts` | `3b5bc05c4563ffa183a4302fe4d4d1af4d6d0038f494d81d0516f5df26d90207` |
| Chat Gun | `backend/src/tools/production-runtime-tool-descriptors.ts` | `c6a914a380678826c924c0c2de9be76cdbfecb2f7eb3b94512369ca1e2ad39d0` |
| Chat Gun | `backend/node_modules/@gun-ai/harness-kernel/package.json` | `19b578698fe72e0c0691ab03530dc4125b207e67c731996ac515dc439d8858c2` |
| Chat Gun | `backend/node_modules/@gun-ai/harness-kernel/src/side-effect.ts` | `b16e1ff750bed61a27c7cd61cfd71ea8904aa57b68f6e9d74be0af7e8b9ffd2f` |
| Chat Gun | `backend/node_modules/@gun-ai/harness-kernel/dist/side-effect.js` | `dca7eb00a3e2c2a048fad57e0159fcf03702920f39e819eaae7d36e92b957e51` |
| Chat Gun | `backend/node_modules/@gun-ai/harness-kernel/src/idempotency.ts` | `f33efc1d547097eae9773a3aa1064a616ee296d1c2da28ac7de8c40b53abeb58` |
| Chat Gun | `backend/node_modules/@gun-ai/harness-kernel/src/retry.ts` | `1a47d1c61fc8af8c3171a4c035b4be031cda7d275ec73513c6256eda4f8f2677` |
| CCB | `src/query.ts` | `74e0ce0d86cfd453add8dc1d15ccb6311b02964b8321e3721b8e71fbd87252ce` |
| CCB | `src/utils/sessionStorage.ts` | `8a123ebce1ee72b9081d34b8f3697e5fcc9c7576df5b98e4206bb28414134412` |
| CCB | `src/utils/sessionRestore.ts` | `9dfe9c2203aac5c3cc6d49e45bf2f8fd381582781b5cec3abbc4ceb6d219e177` |
| Chat Gun | `backend/src/tools/registry.ts` | `7aaa050b83e5668c6344fea7805b0102a57e809f39f25836de411dcc46a276f3` |
| Chat Gun | `backend/src/operations/recovery/worker-recovery.ts` | `bb40d2772b9efd0f721dc1dcd4ac1a25f5fda8bc5da219f1c770cf9658aca62c` |
| Chat Gun | `backend/src/operations/recovery/reaper.ts` | `09e299dbc4285ffc6a41debbeefd00ab6ab42f70631b3d94f38e41b2b2f911b5` |
| Chat Gun | `backend/src/runtime/side-effect/tool-execution-runner.test.ts` | `0f223d5f2c55862b42ddd32c1eec540de2365030361f88b78f09d68e74429a3f` |
| CCB | `src/utils/fileHistory.ts` | `372c430358c09e3c374817f34e2335a9e82a4075e61b2576034f91e11fca011c` |
