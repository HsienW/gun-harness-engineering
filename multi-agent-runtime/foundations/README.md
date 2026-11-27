# Multi-Agent Runtime foundations

[English](./README.md) | [繁體中文](./README-zh-TW.md) | [简体中文](./README-zh-CN.md)

The Munder Difflin main process starts each worker CLI in a pseudoterminal (PTY). Hive manages assignments and messages. Within each turn, the CLI samples from the model and feeds tool results back to it. To trace a run, start with the main process’s record of the worker, then follow its messages to find the work still pending. [Platform architecture](https://github.com/HarnessMD/munder-difflin/blob/96ae43e3292447f68af6904afb353f6f5bdaa412/docs/ARCHITECTURE.md)

1. [How does the platform know which worker it is talking to after a restart?](./01-worker-identity.md) Trace five IDs, a replacement process that reuses a PTY ID, and a Windows startup failure that cut off part of an argument.
2. [When will the worker process a delivered message?](./02-mailbox-and-cli-progress.md) Follow router delivery, the CLI’s mailbox check, the end of a turn, and the reply.
3. [What does the platform need to resume a task after a restart?](./03-recovery-and-state-evidence.md) Check role memory, the transcript, the CLI session, and the task record to see what evidence is still missing.

The source code, design documents, and changelog all refer to the same revision. The source shows how PTY callbacks and session registration work. The Hive design describes the expected Stop hook behavior. The exercises in these articles still need to be run. [Revision 96ae43e](https://github.com/HarnessMD/munder-difflin/commit/96ae43e3292447f68af6904afb353f6f5bdaa412)

[Back to Multi-Agent Runtime (Traditional Chinese)](../README-zh-TW.md)
