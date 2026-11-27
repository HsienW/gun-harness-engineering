# Two workers got different results. Which report can the next worker rely on?

[English](./02-conflicting-worker-reports.md) | [繁體中文](./02-conflicting-worker-reports-zh-TW.md) | [简体中文](./02-conflicting-worker-reports-zh-CN.md)

Worker A writes in the shared notes, “Taiwan address tests passed on revision B.” Later, worker B replies, “Order import still rejects Taiwan addresses on that revision,” and attaches a failure log. Worker C takes over the report. If C goes by the later reply alone, the report will overlook a fix that A verified. If C copies the shared note, the user may think order import passed too.

In the address example from the previous article, suppose the original task has one acceptance criterion: the address validation function must accept the Taiwan address it previously rejected, with house number `12號`. C follows the links in A’s notes and checks the attachment in B’s reply to confirm that both workers tested the same revision. C then checks which address each worker used, where they submitted it, and at which step they observed the result.

| Test input | A’s record | B’s record | What C can report |
|---|---|---|---|
| House number `12號` | A1 called the address validation function directly. It accepted the address. | B1 submitted the same address through order import. The importer reported an invalid address. | On revision B the function accepts this input, and order import rejects it. Both observations belong in the report. The cause of the import failure is still open. |
| Hyphenated house number `12-1號` | A2 created a test order through order import. | B2 used the same import file and got an invalid-address error. | Both runs used order import. The records also agree on the revision, input file, and region settings. The discrepancy still needs investigation. |

A1 answers the acceptance question for the original task. C can give that record to the coordinator and report B1 separately. B1 shows that the import screen returned “invalid address.” That message alone does not tell C which step rejected the address. The next worker can use B1’s import file to check what value the importer actually passed to the validation function. Changing the import module would require a separate decision about the scope of the assignment. [Can another worker’s reply change my task? (Traditional Chinese)](./01-peer-replies-and-task-scope-zh-TW.md)

## Which test is worth rerunning?

A2 and B2 both went through order import, so the case worth rerunning is `12-1號`. Suppose B says “the same file” but attaches only a screenshot of the error. C should ask B for the file used in that run or a file hash to compare. Until then, the note can read, “B2 reports that order import rejected the address. Input file still to be checked.” Matching house numbers leave the rest of the input unverified.

In the example in the table, C has compared the two files and can ask the coordinator to assign this case to the verifier. The verifier checks out revision B in a clean test workspace and runs the saved import file with the region settings from the original records. They record the settings the importer actually loaded, its response, and whether it created an order. If it rejects the address again, the verifier has a reproduction record to investigate. If it creates an order, that success also goes on record. The cause of the rejection in B2 is still open.

## What should the shared note tell the next worker?

Even before the rerun, C can narrow the original claim that “Taiwan address tests passed.” In this example, A1, B1, A2, and B2 each refer to the original record for that run.

> Revision B, Taiwan addresses. The function test for house number `12號` passed as required by the original task (A1). Order import rejected the same address (B1). For house number `12-1號`, order import created an order in A2 and reported an invalid address in B2. The same file produced different results. The verifier will rerun that case.

Suppose the verifier reports that the rerun created an order. C can add this entry beside A2 and B2: “Rerunning the same file on revision B created an order. See R1 for the settings and result. The cause of the rejection in B2 is still open.” The next worker investigating the failure can find the original response in B2. The coordinator checking the address function can go straight to A1.

If the implementer later delivers revision C, the note still documents what happened on revision B. The next worker can reuse the input files and test steps and record the results for C separately. The coordinator can then see which cases were tested on the revision being delivered. [Which assignments still apply when requirements change?](../foundations/04-revising-delegated-work.md)

[Back to Input and Context (Traditional Chinese)](./README-zh-TW.md)
