## Architecture Overview

This system implements a **database-first, SQS-backed distributed task execution architecture** designed for correctness, resilience, and observability.

At a high level:

- **PostgreSQL** is the *source of truth* for task state and lifecycle
- **Amazon SQS** (LocalStack in dev/test) is used for *delivery and wake-up*, not for state
- **Workers** are stateless and idempotent
- **Retries, backoff, and terminal failure** are governed entirely by the database

This design intentionally favors **correctness over convenience** and scales safely to multiple workers.

---

## Core Components

### 1. Tasks Table (PostgreSQL)

Each task is represented as a row in the `tasks` table, with fields covering:

- Lifecycle state (`PENDING`, `ENQUEUED`, `PROCESSING`, `SUCCEEDED`, `FAILED`, `DEAD`)
- Scheduling (`scheduled_for`)
- Retry control (`attempt_count`, `max_attempts`)
- Execution metadata (`worker_id`, `processing_started_at`, `completed_at`)
- Failure diagnostics (`last_error`)

The database enforces **state transitions** and **idempotency guarantees**.

---

### 2. DueTaskEnqueuer (Scheduler → SQS)

A periodic enqueuer:

- Selects tasks that are **due** (`scheduled_for <= now`)
- Acquires a **DB-level lock** to prevent double-enqueue
- Sends the task ID to SQS
- Releases the enqueue lock only after successful send

If enqueue fails, the task remains eligible and will be retried safely.

---

### 3. SQS Worker Loop

Workers operate in a long-polling loop:

1. Receive message from SQS
2. Parse task ID
3. Attempt to **claim the task in the database**
4. Process the task
5. Persist the outcome (success or failure)
6. Delete the SQS message **only after DB commit**

Workers are stateless and can be horizontally scaled.

---

## Why the Database Is the Source of Truth

This architecture **intentionally does not rely on SQS for correctness**.

### Reasons:

- SQS provides **at-least-once delivery**, not exactly-once
- Messages may be duplicated, delayed, or reordered
- Visibility timeouts can expire mid-processing
- Workers can crash at any point

If SQS were the source of truth, these behaviors would cause corruption.

Instead:

- **All authoritative decisions live in the database**
- SQS is treated as a *best-effort delivery signal*
- The DB acts as an idempotency gate and concurrency arbiter

This guarantees correctness even under failure, retries, or duplication.

---

## At-Least-Once Handling (End-to-End)

The system is explicitly designed for **at-least-once delivery semantics**.

### Claim Phase (Idempotency Gate)

Before processing a task, the worker executes:

- An atomic `UPDATE ... WHERE status = 'ENQUEUED' AND scheduled_for <= now()`
- Exactly **one worker** can transition the task to `PROCESSING`
- All other workers fail the claim and safely exit

This ensures:
- Duplicate SQS messages do **not** cause duplicate execution
- Redeliveries are harmless

---

### Success Handling

On successful execution:

- Worker updates the DB (`PROCESSING → SUCCEEDED`)
- Only the worker that claimed the task can succeed it
- **Only after DB success** is the SQS message deleted

If the DB update fails, the message is *not deleted* and will be retried.

---

### Failure & Retry Handling

On failure:

- Worker records the error in the DB
- If retries remain:
  - Task is rescheduled via `scheduled_for = now + backoff`
  - Status transitions back to `ENQUEUED`
- If retries are exhausted:
  - Task becomes `DEAD`

Again:
- SQS message is deleted **only if the DB update succeeds**
- Otherwise, the message is allowed to redeliver safely

---

## Key Design Guarantees

This architecture guarantees:

- ✅ No task is lost
- ✅ No task is processed twice successfully
- ✅ Crashes are safe at any point
- ✅ Retries are explicit and observable
- ✅ Workers are horizontally scalable
- ✅ Behavior is fully testable end-to-end

---

## Design Philosophy

> **SQS delivers messages.  
> PostgreSQL decides reality.**

By separating *delivery* from *truth*, the system achieves strong correctness guarantees while still benefiting from SQS scalability and simplicity.

This pattern closely mirrors real-world production systems used at scale for distributed job execution.
