# 📅 Day 16 Log — Failure Handling, DLQ, and Production-Grade Metrics

## Focus
Primary goal today was to **complete the failure lifecycle** for SQS-backed tasks and make it *observable, testable, and production-correct*.

---

## ✅ What I Built Today

### 1. **Manual DLQ Publishing**
I implemented **explicit DLQ publishing from the worker** instead of relying on SQS redrive policies.

**Why this matters**
- Keeps **Postgres as the source of truth**
- DLQ events are tied to **domain state (DEAD)**, not delivery attempts
- Allows richer DLQ payloads (task metadata, attempts, error)

**Flow**
1. Task fails during processing
2. DB update via `markFailedAndRescheduleOutcome(...)`
3. If `becameDead == true`:
   - Publish structured `DeadTaskEvent` to DLQ
   - Delete original SQS message (avoid poison loops)

---

### 2. **DeadLetterClient + SqsDeadLetterClient**
Introduced a clean abstraction for DLQ publishing:

- `DeadLetterClient` interface
- `SqsDeadLetterClient` implementation
- JSON payload includes:
  - taskId
  - workerId
  - attemptCount / maxAttempts
  - error
  - timestamp

This keeps the worker logic clean and testable.

---

### 3. **New Metric: `dtp_tasks_dead_lettered_total`**
Added a **high-signal production metric**:
dtp_tasks_dead_lettered_total

**Semantics**
- Incremented **exactly once per task**
- Triggered when DB transitions task → `DEAD`
- Independent of DLQ publish success/failure

This enables:
- Alerting on poison-task accumulation
- Correlation with retry failures
- Clear operational visibility

---

### 4. **Worker Logic (Finalized)**
The SQS worker now correctly handles all cases:

- Invalid messages → delete immediately
- Claim failure → do not delete (retry safe)
- Success → DB update → delete
- Failure → DB update → optional DLQ publish → delete
- DLQ publish failure → still delete original message (chosen policy)

This guarantees:
- No infinite retry loops
- No task loss
- At-least-once processing with DB authority

---

### 5. **Tests (Unit + Integration)**
All relevant tests are now passing:

#### Unit Tests
- Success path
- Retry path
- Dead task path
- DLQ publish success & failure
- Correct delete/no-delete behavior
- Metrics verification

#### Integration Test (LocalStack + Postgres)
- End-to-end:
  - Task enqueued
  - SQS consumed
  - DB transitions to `SUCCEEDED` or `DEAD`
  - DLQ receives exactly one message for poison task

This validates the *real system*, not mocks.

---

## 🧠 Key Design Decisions 

- **DB as source of truth**, SQS as transport
- **Manual DLQ publishing** for domain-aligned failure handling
- **Outcome-based DB updates** (`FailOutcome`)
- **Metrics tied to state transitions**, not infra success
- **Delete SQS messages only after DB success**

---

## 📈 Why This Is Worthy

This work demonstrates:
- Distributed systems thinking
- Failure semantics beyond “retry”
- Production observability
- Clear tradeoff reasoning (DLQ strategy A vs B)
- Strong testing discipline (unit + integration)

