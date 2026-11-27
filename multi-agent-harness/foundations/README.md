# Multi-Agent Harness foundations

[English](./README.md) | [繁體中文](./README-zh-TW.md) | [简体中文](./README-zh-CN.md)

An assignment tells a worker which workspace to use and whom to reply to. It also gives whoever reviews the work a way to identify the deliverable. These articles use Munder Difflin’s Hive collaboration protocol as their example.

1. [How do you delegate work to another agent?](./01-delegation-contract.md) Specify the recipient, scope, and reporting requirements for a README check.
2. [Who decides whether the work is complete once the replies arrive?](./02-delivery-and-acceptance.md) Match the implementation and verification reports to the same revision and decide whether more evidence is needed.
3. [How should you divide work among workers?](./03-work-decomposition.md) Decide what can run in parallel, who needs an upstream deliverable, and when one worker is enough.
4. [Which assignments still apply when requirements change?](./04-revising-delegated-work.md) Follow a change in scope to identify affected assignments, reusable artifacts, and revisions that need further testing.

The message and reply references come from a pinned Munder Difflin revision. The division of work, revisions B and C, reassignments, and acceptance decisions are teaching examples. [Revision 96ae43e](https://github.com/HarnessMD/munder-difflin/commit/96ae43e3292447f68af6904afb353f6f5bdaa412) · [Hive collaboration protocol](https://github.com/HarnessMD/munder-difflin/blob/96ae43e3292447f68af6904afb353f6f5bdaa412/HIVE.md)

[Back to Multi-Agent Harness (Traditional Chinese)](../README-zh-TW.md)
