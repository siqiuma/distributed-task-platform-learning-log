## Day 7 – Lesson 7 (Part 1): Retry Logic, Failure Handling, and Domain Robustness

### What I worked on
- Extended the `Task` domain model to support retry semantics.
- Added retry-related fields to the database schema using Flyway (`V3__add_retry_fields.sql`).
- Introduced `attempt_count`, `max_attempts`, `next_run_at`, and `last_error` columns to track task execution history.
- Debugged and resolved schema validation issues between Hibernate and Postgres during the migration.
- Fixed column type mismatches caused by using `@Lob` vs `TEXT` in Postgres.
- Explicitly mapped new columns using `@Column(name = ...)` to keep JPA and the database schema aligned.
- Verified schema changes directly in Postgres using `psql` (`\d tasks`).
- Investigated and resolved IntelliJ warnings related to JPA schema inspection and Flyway-managed tables.

### What I learned
- **Retries are a first-class domain concern**
  - Failures are expected in distributed systems and should be handled intentionally.
  - Retry behavior must be bounded to avoid infinite loops and resource exhaustion.
- **Why retry state belongs in the domain**
  - `attemptCount` and `maxAttempts` make retry limits explicit and enforceable.
  - `lastError` provides visibility into why a task failed, which is critical for debugging and observability.
  - `nextRunAt` allows retries to be delayed or backoff strategies to be implemented later.
- **Flyway and schema evolution**
  - Schema changes must be additive and versioned once a database is persistent.
  - Editing old migrations is unsafe once they’ve been applied; fixes require new migrations.
- **Postgres vs Hibernate expectations**
  - Postgres `TEXT` is preferable to `@Lob` for storing large strings like error messages.
  - Using `@Lob` can cause Hibernate to expect `OID/CLOB`, leading to validation errors.
- **IDE vs runtime reality**
  - IntelliJ JPA inspections can flag false errors if it is not connected to or synced with the actual database.
  - Runtime correctness (Flyway + Postgres) takes precedence over IDE warnings.
- **Why explicit column mappings help**
  - Adding `@Column(name = "...")` removes ambiguity between Java naming conventions and SQL column names.
  - This improves clarity and reduces tooling confusion when schemas are managed externally.

### Decision(s) made
- Use Postgres `TEXT` columns for `payload` and `last_error` instead of `@Lob`.
- Treat retry tracking as part of the core domain model, not worker-only logic.
- Keep Flyway as the single source of truth for schema evolution.
- Prefer explicit JPA column mappings to avoid reliance on naming strategy inference.

### Problems / blockers
- Encountered Hibernate schema validation errors due to mismatched expectations for `last_error` column type.
- IntelliJ flagged unresolved column warnings even though the application compiled and ran successfully.
- Required manual database inspection and IDE refresh to confirm correctness.

### Open questions / next steps
- Update worker logic to:
  - increment `attemptCount` on each processing attempt
  - reschedule tasks using `nextRunAt`
  - mark tasks permanently `FAILED` when retries are exhausted
- Add repository queries that fetch only runnable tasks (`status = PENDING` and `next_run_at <= now`).
- Decide whether to introduce exponential backoff or fixed-delay retry strategies.
- Continue with the remaining steps of Lesson 6 before writing the final Lesson 6 summary.
