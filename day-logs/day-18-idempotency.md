# Day18 — Idempotency & Exactly-Once Semantics

**Lesson Type:** Core distributed systems correctness  

---

## Goal of This Lesson

Ensure that **each task’s side effects are applied exactly once**, even though:

- SQS provides **at-least-once delivery**
- Messages may be **redelivered**
- Multiple workers may **race**
- Workers may **crash after partial execution**

The system must be **safe under retries and concurrency**.

---

## Why This Matters

In real distributed systems:

> *Exactly-once delivery is impossible.*  
> *Exactly-once effects are achievable.*

This lesson demonstrates how to **separate delivery guarantees from processing guarantees**, which is a classic senior-level systems design concept.

---

## Design Overview

### Key Idea

Use **database-backed idempotency** to guard side effects.

Even if:
- The same SQS message is delivered multiple times
- Multiple workers receive the same task
- A worker crashes mid-processing

➡️ The task’s effect is applied **once and only once**.

---

## Components Added

### 1️⃣ Idempotency Table

```sql
CREATE TABLE task_idempotency (
    task_id BIGINT PRIMARY KEY,
    processed_at TIMESTAMPTZ NOT NULL
);
```
-   One row per successfully completed task

-   Primary key ensures **only one winner**

-   Database enforces correctness (not the application)
### 2️⃣ Idempotency Repository

`public interface TaskIdempotencyRepository {
    boolean alreadyProcessed(long taskId);
    boolean recordProcessed(long taskId);
}`

**Postgres upsert pattern:**

`INSERT INTO task_idempotency(task_id, processed_at)
VALUES (?, now())
ON CONFLICT (task_id) DO NOTHING;`

-   First worker inserts successfully

-   All others are rejected automatically

* * * * *

Worker Execution Flow (Updated)
-------------------------------
1\. Receive SQS message

2\. Claim task in DB (ENQUEUED → PROCESSING)

3\. Check idempotency table

   ├─ already processed → skip safely

   └─ not processed → continue

4\. Execute business logic

5\. Record idempotency

6\. Mark task SUCCEEDED

7\. Delete SQS message
* * * * *

Exactly-Once Effect Guarantee
-----------------------------

| Failure Scenario | Outcome |
| --- | --- |
| Message redelivery | Safe (idempotency table blocks duplicate work) |
| Multiple workers racing | Only one records idempotency |
| Crash after DB commit | Retry sees idempotency → no duplicate |
| Crash before DB commit | Task retried safely |

**Guarantee:**

> A task's side effects are applied **at most once**, even under retries.

* * * * *

End-to-End Validation
---------------------

### Test: `e2e_threeWorkers_noDoubleProcessing_allSucceeded`

-   Insert **N tasks**

-   Start **3 worker instances**

-   Workers poll concurrently

-   Assertions:

    -   Every task processed **exactly once**

    -   No duplicate side effects

    -   All tasks reach `SUCCEEDED`

`assertThat(processedCounts.get(taskId).get()).isEqualTo(1);`

This test proves:

-   Correct concurrency handling

-   Correct idempotency enforcement

-   Production-grade behavior

* * * * *

Key Lessons Learned
-------------------

-   Exactly-once **delivery** is not required for correctness

-   Databases are powerful coordination mechanisms

-   Idempotency must live **outside** business logic

-   Correctness > performance optimizations

-   Tests must simulate real concurrency to be meaningful

* * * * *

Guarantees Achieved
-------------------

✔ At-least-once message delivery\
✔ Exactly-once task execution effects\
✔ Safe retries after crashes\
✔ Horizontal scalability without duplication
