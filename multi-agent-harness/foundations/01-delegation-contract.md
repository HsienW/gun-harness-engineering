# How do you delegate work to another agent?

[English](./01-delegation-contract.md) | [繁體中文](./01-delegation-contract-zh-TW.md) | [简体中文](./01-delegation-contract-zh-CN.md)

Suppose a coordinator needs a worker to find a broken link in a README. If the assignment just says “check the docs,” a reply of “checked” leaves the recipient guessing which revision the worker inspected. Specify the workspace, revision, and file, and ask for the line number and target path in the reply. Someone else can then repeat the check.

| What to specify | Agreement for this check |
|---|---|
| Who assigns the work and who receives it? | The coordinator creates the task, names the worker, and keeps the message ID of the original assignment. |
| What may the worker do? | Read the README at the specified revision and resolve relative paths from the file’s directory. If a file needs editing, report that first so the coordinator can confirm the change in scope. |
| How should the worker report back? | Reply through the outbox, use `in_reply_to` to identify the original message, and include the revision, line number, and broken path. |
| Who closes the task? | The coordinator opens the file at the specified revision, checks the link target, and records the acceptance result on the task. |

## Did the worker receive the collaboration instructions?

A worker needs instructions for using the inbox and outbox to read an assignment and reply to the original message. In Munder Difflin, a worker on Windows could start successfully even though those instructions had been cut off. To check the full handoff, the coordinator can send a request with a recognizable ID and verify that the worker’s reply points back to it. The startup failure and its process-level evidence are described in [Worker startup and identity (Traditional Chinese)](../../multi-agent-runtime/foundations/01-worker-identity-zh-TW.md).

Hive distinguishes requests that need an answer from notifications that can end an exchange. When the coordinator changes the recipient or asks for a different revision, the assignment should name the new owner and target so the reviewer can tell which reports still apply. The table gives the terms of this example. The protocol defines the actual message fields. [Hive collaboration protocol](https://github.com/HarnessMD/munder-difflin/blob/96ae43e3292447f68af6904afb353f6f5bdaa412/HIVE.md)

[Back to Multi-Agent Harness foundations](./README.md)
