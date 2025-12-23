## Day 6 – Concurrency, Optimistic Locking, and Transaction Boundaries

### What I worked on
- Implemented optimistic locking on the `Task` entity using `@Version` to prevent silent concurrent overwrites.
- Refactored the background worker to use an explicit **claim-first** pattern (`PENDING → PROCESSING`) via a conditional update query.
- Split long-running task processing from database transactions to avoid holding locks during simulated work.
- Introduced a dedicated transactional helper (`TaskWorkerTx`) to ensure short, well-scoped transactions and avoid Spring self-invocation issues. *[tx stands for transactional boundary (or “transaction helper”). It means All database work that must happen inside a transaction lives in this helper.]*
- Investigated and resolved H2 database lock timeout errors caused by long-running operations overlapping with concurrent updates.
- Reviewed worker vs. cancel endpoint interactions under concurrency to ensure correct behavior.

### What I learned
- Optimistic locking (`@Version`) detects stale updates by enforcing `id + version` matching during updates, turning silent data corruption into explicit failures.
- Fixing one concurrency issue (double-claim race) can surface deeper issues (transaction duration and database locking).
- Even without `@Transactional` on a scheduled method, repository operations still run inside transactions.
- Holding a database transaction open while performing slow work (e.g., `Thread.sleep`) can cause lock contention and timeouts, especially in H2.
- The correct pattern for async workers is:
  1. Claim work atomically in a short transaction
  2. Perform processing outside any transaction
  3. Persist results in another short transaction
- `@Transactional(readOnly = true)` disables dirty checking, communicates intent, and improves safety and performance for read operations.

### Decision(s) made
- Use **conditional update queries** (status-based compare-and-set) instead of entity mutation for task state transitions where possible.
- Keep all worker-related database interactions short and transactional, and move processing logic outside transaction boundaries.
- Accept optimistic lock conflicts as a normal outcome in concurrent systems and handle them gracefully (skip/log instead of failing).
- Continue using H2 for learning but plan to migrate to Postgres for more production-realistic locking behavior.

### Problems / blockers
- Encountered repeated `ObjectOptimisticLockingFailureException` and H2 table lock timeout errors while testing concurrent worker and API updates.
- Initially confusing distinction between “race condition fixed” and “lock contention still possible” required deeper understanding of transaction scope.

### Open questions / next steps
- Migrate the project to Postgres (via Docker) to observe more realistic concurrency behavior.
- Convert remaining entity-based updates to conditional update queries for consistency.
- Add retry/backoff or idempotent handling for worker failures.
- Update system diagrams to reflect the claim-and-process worker model.
- Write automated tests that simulate concurrent worker and cancel scenarios.
