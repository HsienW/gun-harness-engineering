# Behavior and Capabilities

[English](./README.md) | [繁體中文](./README-zh-TW.md)

This section follows [Input and Context](../input-and-context/README.md). It traces effective instructions, capability composition, candidate selection, and the handoff from a proposed call to action governance.

Four article drafts are available in English and Traditional Chinese.

## Reading order

| Article | Focus |
|---|---|
| [01. Instructions and behavior](./01-instructions-and-behavior.md) | Prompt replacement and appending, Skill context changes, and the places where code consumes behavior rules. |
| [02. Capability composition](./02-capability-composition.md) | Tool pools, Skills and Commands, MCP loading, and Chat Gun's SDK descriptor staging. |
| [03. Capability selection](./03-capability-selection.md) | Visibility filters, child Agent scope, code routing, model proposals, and Provider compatibility. |
| [04. A capability use case](./04-case-capability-use.md) | A calculation traced through the MCP Agent and compared with the Math Agent's direct calculator call. |

## Sources and evidence

GUN-ATLAS v1.0 supplied the repository index. The relevant source was checked again through GitHub and pinned to these commits:

| Source | Branch inspected | Pinned commit |
|---|---|---|
| [HsienW/chat-gun](https://github.com/HsienW/chat-gun) | `master` | [`92a5673003e4ac4ec09be8b1b350f42dbf4a5e69`](https://github.com/HsienW/chat-gun/commit/92a5673003e4ac4ec09be8b1b350f42dbf4a5e69) |
| [claude-code-best/claude-code](https://github.com/claude-code-best/claude-code) | `main` | [`77a7934e15d69da13879112ed7db695c9ee7a52a`](https://github.com/claude-code-best/claude-code/commit/77a7934e15d69da13879112ed7db695c9ee7a52a) |

CCB means the third-party Claude Code Best repository named in the attachment. Its behavior is described as an observed implementation of that project, without treating it as current official Anthropic behavior.

The drafts link to source at the pinned commits. This work inspected code and call sites; it did not run either repository's unit, integration, E2E, live Provider, or MCP tests. The calculation is a source walkthrough, not a captured execution. Test descriptions identify the code that was read, while deployment behavior and actual outcomes remain unverified. Design proposals are marked separately.

The refreshed Chat Gun source includes `current_weather` SDK descriptor staging in the Registry. The MCP Agent binds a loaded tool list to the model; the Math Agent can extract an expression and call the calculator directly. The Registry's `source` selects dispatch pipeline flags, which does not establish per-request Principal or Tenant filtering of local tool visibility.

## Scope

Input and Context establishes task material, sources, and budget. This section follows the assembly and consumption of instructions and capabilities. Action Governance expands authorization, approval, retries, and side effects for a concrete call. Interaction and Feedback covers interruption and recovery state.

See the [Traditional Chinese guide](./README-zh-TW.md) for the architecture diagram, source map, and detailed evidence boundaries.
