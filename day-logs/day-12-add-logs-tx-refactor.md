### Day 12 — Lesson 11 Adding Logs + Transaction Refactor

#### Scope
Added logs and refactored worker/transaction boundaries toward a more production-ready design.

---

### What Was Completed

#### Transaction boundary refactor
- Introduced **`TaskSnapshot`** as a read-only projection for worker logic.
- Updated `TaskWorkerTx` to expose:
  - `claimAndGetSnapshot(Long id)` — atomic claim + snapshot
  - `markSucceeded(Long id)` → returns updated snapshot
  - `markFailed(Long id, Exception e)` → returns updated snapshot
- Clarified responsibility split:
  - **Repository**: persistence + queries
  - **TaskWorkerTx**: transactional state transitions
  - **TaskWorker**: orchestration only

---

#### Repository updates
- Added `findSnapshotById(Long id)` JPQL projection query.
- Kept `claim(id, now)` as a modifying query to preserve atomic row-claim semantics.

---

#### Test updates
- Migrated “claim exactly once” verification to **`TaskWorkerTxTest`**:
  - Validates `claimAndGetSnapshot()` returns a snapshot once, then `null`.
- Adjusted repository and worker tests to assert via:
  - Snapshots for transactional correctness
  - Repository reads only where necessary
- All tests pass under Testcontainers + Postgres.

---

### Observability improvements (partial)
- Improved worker logs to be **structured and intent-driven**:
  - `task_claimed`
  - `task_processing_started`
  - `task_succeeded`
  - `task_processing_failed_expected`
  - `task_processing_failed_unexpected`
  - `task_retry_state_updated`
  - `task_conflict_optimistic_lock`
- Distinguished **expected failures (WARN)** vs **unexpected failures (ERROR)**.
- Added duration metrics (`durationMs`) to logs for latency visibility.

---

### Key Takeaways
- Worker correctness is now **provable via tests**, not assumptions.
- Retry logic is enforced at both **domain** and **worker** layers.
- Transaction boundaries are explicit and interview-explainable.
- Logs now tell a coherent “task lifecycle story”.

---

### Next Step
➡️ finalize observability:
- Metrics (latency, success/failure rates)
- Meaningful health/readiness checks
- Tie logs + metrics into an interview-ready narrative
