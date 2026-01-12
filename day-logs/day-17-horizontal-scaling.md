# Day 17 — Horizontal Scaling with Safe Task Claiming

**Goal:**  
Prove that multiple worker instances can process tasks concurrently **without double execution**, using the database as the single source of truth.

---

## Why This Lesson Matters

Horizontal scaling is one of the most common system design interview topics.

- Can you safely run **multiple workers**?
- How do you prevent **double processing**?
- What happens under race conditions?
- How do you prove correctness?

This lesson turns the system from a “single-worker demo” into a **production-grade distributed worker system**.

---

## Design Principles

### 1. Database Is the Source of Truth
SQS provides **at-least-once delivery**, not exactly-once.  
Therefore:
- SQS **does not guarantee uniqueness**
- The database **must decide ownership**

All task ownership decisions are enforced via an **atomic UPDATE** in PostgreSQL.

---

### 2. Atomic Claiming via SQL
Each worker attempts to claim a task using:

```sql
UPDATE tasks
SET status = 'PROCESSING',
    worker_id = ?,
    attempt_count = attempt_count + 1,
    processing_started_at = now(),
    updated_at = now()
WHERE id = ?
  AND status = 'ENQUEUED'
  AND (scheduled_for IS NULL OR scheduled_for <= now())
  AND attempt_count < max_attempts;
```
**Guarantee:**\
Only **one worker** can update a given row → only one worker owns the task.

* * * * *

### 3\. Worker Behavior

Each worker follows the same deterministic lifecycle:

1.  Receive SQS message (taskId)

2.  Attempt DB claim

    -   If claim fails → do nothing (another worker won)

3.  Execute task logic

4.  Persist final state (`SUCCEEDED`, `FAILED`, or `DEAD`)

5.  Delete SQS message **only after DB update succeeds**

* * * * *

Horizontal Scaling Test
-----------------------

### Test Setup

-   3 independent `SqsWorkerLoop` instances

-   Shared PostgreSQL database

-   Shared SQS queue (LocalStack)

-   All workers started **simultaneously**

`SqsWorkerLoop w1 = new SqsWorkerLoop(...);
SqsWorkerLoop w2 = new SqsWorkerLoop(...);
SqsWorkerLoop w3 = new SqsWorkerLoop(...);`

Each worker repeatedly calls:

`worker.runOnce();`

until all tasks are completed.

* * * * *

### Assertions

The integration test verifies:

`SELECT COUNT(*)
FROM tasks
WHERE status <> 'SUCCEEDED'
  AND id = ANY(?);`

**Expected result:** `0`

Meaning:

-   Every task completed successfully

-   No task was skipped

-   No task was processed twice

* * * * *

Observed Behavior (Logs)
------------------------

-   Multiple workers received the **same SQS messages**

-   Only one worker successfully claimed each task

-   Other workers logged:

    `Task not claimed (already claimed or not due)`

-   No duplicate processing occurred

-   System converged correctly under concurrency

* * * * *

Guarantees Achieved
-------------------

| Guarantee | Achieved |
| --- | --- |
| At-least-once delivery | ✅ (SQS) |
| Exactly-once task ownership | ✅ (DB atomic claim) |
| No double processing | ✅ |
| Horizontal scalability | ✅ |
| Crash-safe retry | ✅ |

* * * * *

What This Proves
----------------

-   Workers are **stateless**

-   Scaling is achieved by **adding more workers**, not changing logic

-   Correctness does **not depend on SQS ordering**

-   Database enforces ownership deterministically
