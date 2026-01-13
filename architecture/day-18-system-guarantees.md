## Guarantees & Design Decisions

This system is designed to behave correctly under **failures, retries, and concurrency**, using well-defined guarantees rather than relying on fragile assumptions.

---

### Delivery vs Execution Guarantees

| Concern | Guarantee |
|-------|----------|
| Message delivery | **At-least-once** (SQS) |
| Task execution | **At-most-once effects** |
| Overall outcome | **Exactly-once side effects** |

> We do **not** rely on exactly-once delivery.  
> Instead, we enforce correctness at the execution layer.

---

## Source of Truth: Database

The **database is the single source of truth** for task lifecycle.

All important state transitions happen in Postgres:
- Task claiming
- Retry counting
- Success / failure recording
- Dead-lettering
- Idempotency enforcement

Queues are treated as **transport**, not state.

---

## Task Ownership & Concurrency

Multiple workers may receive the same SQS message concurrently.

Only **one worker can own a task**, enforced by an atomic update:

```sql
UPDATE tasks
SET status = 'PROCESSING',
    worker_id = ?,
    attempt_count = attempt_count + 1,
    processing_started_at = now()
WHERE id = ?
  AND status = 'ENQUEUED'
  AND (scheduled_for IS NULL OR scheduled_for <= now())
  AND attempt_count < max_attempts;
```
**Why this works:**

-   The update is atomic

-   Exactly one row can match

-   All other workers fail the claim and back off

No distributed locks are required.

* * * * *

Idempotency & Exactly-Once Effects
----------------------------------

SQS provides **at-least-once delivery**, so messages may be redelivered.

To prevent duplicate side effects, the system uses a **database-backed idempotency table**:
```sql
CREATE TABLE task_idempotency (
    task_id BIGINT PRIMARY KEY,
    processed_at TIMESTAMPTZ NOT NULL
);
```

Processing logic:

-   First worker inserts `task_id` successfully

-   All subsequent attempts are rejected automatically

-   Duplicate execution is safely skipped

This guarantees **exactly-once execution effects**, even under retries or crashes.

* * * * *

Failure Handling & Retries
--------------------------

### Retry Semantics

-   Failed tasks are rescheduled with backoff

-   Attempt count is incremented atomically

-   Retries continue until `max_attempts` is reached

### Dead Lettering

-   Tasks exceeding `max_attempts` are marked `DEAD`

-   A dead-task event is published to the DLQ

-   The original SQS message is deleted to avoid poison loops

* * * * *

Message Deletion Rule (Critical)
--------------------------------

> **A message is deleted from SQS only after the database has been updated successfully.**

This ensures:

-   No lost work

-   Safe retries on transient failures

-   Database state always reflects reality

* * * * *

Worker Model
------------

-   Each application instance runs **one worker loop**

-   Horizontal scaling is achieved by running multiple instances

-   No shared memory or coordination required

-   Database guarantees correctness

* * * * *

Long Polling Strategy
---------------------

Workers use **SQS long polling**:

-   Reduces empty receives

-   Lowers AWS cost

-   Prevents CPU busy loops

-   Improves latency under load

* * * * *

What This Design Avoids
-----------------------

❌ Distributed locks\
❌ Exactly-once message delivery assumptions\
❌ In-memory coordination\
❌ Tight coupling between queue and state

* * * * *

Summary
-------

This system guarantees:

✔ Correct execution under concurrency\
✔ Safe retries after crashes\
✔ No duplicate side effects\
✔ Horizontal scalability\
✔ Production-grade fault tolerance

These guarantees mirror real-world distributed task systems used in industry.
